##  The Basics
A already fixed issue I found when code auditing the old version of mali driver.

## The vulnerability
There is an ioctl command `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` in the ioctl of mali driver:
```
1	case KBASE_IOCTL_KCPU_QUEUE_ENQUEUE:
2		KBASE_HANDLE_IOCTL_IN(KBASE_IOCTL_KCPU_QUEUE_ENQUEUE,
3				kbasep_kcpu_queue_enqueue,
4				struct kbase_ioctl_kcpu_queue_enqueue,
5				kctx);
6		break;
```
We can use the command `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` to enqueue a command into the KCPU queue. For example, we can use `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` to enqueue a queue command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT`.

The command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT` is processed in two funtions mainly: the function `kbase_kcpu_map_import_prepare` and the function `kcpu_queue_process`. The function `kbase_kcpu_map_import_prepare` will be called first, and then the function `kcpu_queue_process`.

First of all, let's have a look at the function `kbase_kcpu_map_import_prepare`. The function `kbase_kcpu_map_import_prepare` is used to do some preparation work for the command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT`.
```
1static int kbase_kcpu_map_import_prepare(
2		struct kbase_kcpu_command_queue *kcpu_queue,
3		struct base_kcpu_command_import_info *import_info,
4		struct kbase_kcpu_command *current_command)
5{
6	struct kbase_context *const kctx = kcpu_queue->kctx;
7	struct kbase_va_region *reg;
8	int ret = 0;
9
10	lockdep_assert_held(&kctx->csf.kcpu_queues.lock);
11
12	/* Take the processes mmap lock */
13	down_read(kbase_mem_get_process_mmap_lock());
14	kbase_gpu_vm_lock(kctx);
15
16	reg = kbase_region_tracker_find_region_enclosing_address(kctx,
17					import_info->handle);
18
19	......
20
21	if (reg->gpu_alloc->type == KBASE_MEM_TYPE_IMPORTED_USER_BUF) {
22		......
23		ret = kbase_jd_user_buf_pin_pages(kctx, reg);  //<-----------------------  enter the routine !!!
24		if (ret)
25			goto out;
26	}
27
28	current_command->type = BASE_KCPU_COMMAND_TYPE_MAP_IMPORT;
29	current_command->info.import.gpu_va = import_info->handle;
30
31out:
32	......
33	return ret;
34}
```
As you can see, in the function `kbase_kcpu_map_import_prepare`, the function `kbase_jd_user_buf_pin_pages` gets called to get the associated user pages of the `reg->gpu_alloc`. Let's have a look at the function `kbase_jd_user_buf_pin_pages`:
```
1int kbase_jd_user_buf_pin_pages(struct kbase_context *kctx,
2		struct kbase_va_region *reg)
3{
4	struct kbase_mem_phy_alloc *alloc = reg->gpu_alloc;
5	struct page **pages = alloc->imported.user_buf.pages;
6	unsigned long address = alloc->imported.user_buf.address;
7	struct mm_struct *mm = alloc->imported.user_buf.mm;
8	long pinned_pages;
9	long i;
10
11	if (WARN_ON(alloc->type != KBASE_MEM_TYPE_IMPORTED_USER_BUF))
12		return -EINVAL;
13
14	if (alloc->nents) {
15		if (WARN_ON(alloc->nents != alloc->imported.user_buf.nr_pages))
16			return -EINVAL;
17		else
18			return 0;
19	}
20
21	......
22	pinned_pages = get_user_pages_remote(mm,
23			address,
24			alloc->imported.user_buf.nr_pages,
25			reg->flags & KBASE_REG_GPU_WR ? FOLL_WRITE : 0,
26			pages, NULL, NULL);
27......
28	alloc->nents = pinned_pages; //<--------------  here, initialize the "nents" !!!
29
30	return 0;
31}

```
As you can see the `nents` field of the `reg->gpu_alloc` gets initialized to the `pinned_pages` which represents the number of actual user pages pinned.  The `nents` field of `reg->gpu_alloc`  always represents the actual number of physical pages in this `gpu_alloc`.
So for now, we have understood all the work done by the function `kbase_kcpu_map_import_prepare`. But we need to notice that only the `nents` field gets initialized for now, the `pages` field of the `reg->gpu_alloc` is  not initialized yet. Actually, the `pages` field of the `reg->gpu_alloc` is initialized in the function `kcpu_queue_process`.

Let's have a look at how the function `kcpu_queue_process` processes the command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT`:
```
1		case BASE_KCPU_COMMAND_TYPE_MAP_IMPORT: {
2			struct kbase_ctx_ext_res_meta *meta = NULL;
3
4			KBASE_TLSTREAM_TL_KBASE_KCPUQUEUE_EXECUTE_MAP_IMPORT_START(
5				kbdev, queue);
6
7			kbase_gpu_vm_lock(queue->kctx);
8			meta = kbase_sticky_resource_acquire( //<---------------------  enter the routine !!!
9				queue->kctx, cmd->info.import.gpu_va);
10			kbase_gpu_vm_unlock(queue->kctx);
11
12			if (meta == NULL) {
13				queue->has_error = true;
14				dev_warn(kbdev->dev,
15						"failed to map an external resource\n");
16			}
17
18			KBASE_TLSTREAM_TL_KBASE_KCPUQUEUE_EXECUTE_MAP_IMPORT_END(
19				kbdev, queue, meta ? 0 : 1);
20			break;
21		}

```
As you can see the function `kbase_sticky_resource_acquire` gets called to process the command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT`. Following the function `kbase_sticky_resource_acquire`, we can see the function `kbase_map_external_resource` get called finally:
```
1struct kbase_mem_phy_alloc *kbase_map_external_resource(
2		struct kbase_context *kctx, struct kbase_va_region *reg,
3		struct mm_struct *locked_mm)
4{
5	int err;
6
7	lockdep_assert_held(&kctx->reg_lock);
8
9	/* decide what needs to happen for this resource */
10	switch (reg->gpu_alloc->type) {
11	case KBASE_MEM_TYPE_IMPORTED_USER_BUF: {
12		if ((reg->gpu_alloc->imported.user_buf.mm != locked_mm) &&
13		    (!reg->gpu_alloc->nents))
14			goto exit;
15
16		reg->gpu_alloc->imported.user_buf.current_mapping_usage_count++;
17		if (reg->gpu_alloc->imported.user_buf
18			    .current_mapping_usage_count == 1) {
19			err = kbase_jd_user_buf_map(kctx, reg);  //<------------------------------------ enter the routine !!!
20			if (err) {
21				reg->gpu_alloc->imported.user_buf.current_mapping_usage_count--;
22				goto exit;
23			}
24		}
25	}
26	break;
27	......
28	default:
29		goto exit;
30	}
31
32	return kbase_mem_phy_alloc_get(reg->gpu_alloc);
33exit:
34	return NULL;
35}
```
As you can see, the function `kbase_jd_user_buf_map` gets called:
```
1static int kbase_jd_user_buf_map(struct kbase_context *kctx,
2		struct kbase_va_region *reg)
3{
4	long pinned_pages;
5	struct kbase_mem_phy_alloc *alloc;
6	struct page **pages;
7	struct tagged_addr *pa;
8	long i;
9	unsigned long address;
10	struct device *dev;
11	unsigned long offset;
12	unsigned long local_size;
13	unsigned long gwt_mask = ~0;
14	int err = kbase_jd_user_buf_pin_pages(kctx, reg);
15
16	if (err)
17		return err;
18
19	alloc = reg->gpu_alloc;
20	pa = kbase_get_gpu_phy_pages(reg); //<-------------- get the "reg->gpu_alloc->pages" !!!
21	address = alloc->imported.user_buf.address;
22	pinned_pages = alloc->nents;
23	pages = alloc->imported.user_buf.pages;
24	dev = kctx->kbdev->dev;
25	offset = address & ~PAGE_MASK;
26	local_size = alloc->imported.user_buf.size;
27
28	for (i = 0; i < pinned_pages; i++) {
29		dma_addr_t dma_addr;
30		unsigned long min;
31
32		min = MIN(PAGE_SIZE - offset, local_size);
33		dma_addr = dma_map_page(dev, pages[i],
34				offset, min,
35				DMA_BIDIRECTIONAL);
36		if (dma_mapping_error(dev, dma_addr))
37			goto unwind;
38
39		alloc->imported.user_buf.dma_addrs[i] = dma_addr;
40		pa[i] = as_tagged(page_to_phys(pages[i]));  //<----------------------- initialize the "reg->gpu_alloc->pages" here !!!!
41
42		local_size -= min;
43		offset = 0;
44	}
45......
46	return err;
47}
```
As you can see that the `pages` field of the `reg->gpu_alloc` gets initialized in the function `kbase_jd_user_buf_map`.

So for now, we already know that the `nents` filed of the `reg->gpu_alloc` is initialized in the function `kbase_kcpu_map_import_prepare`, and the `pages` field of the `reg->gpu_alloc` is initialized in the function `kcpu_queue_process`. This situation seems safe. But if we think more, we can find this will lead to a really strange situation: the `nents` field of `reg->gpu_alloc` gets initialized, while the `pages` field of `reg->gpu_alloc` is left uninitialized !!!

We all know that when we mmap() the  `reg->gpu_alloc` to CPU side, it only checks whether the `nents` filed of the `reg->gpu_alloc` is non-zero. And if the the `nents` filed of the `reg->gpu_alloc` is non-zero, then we can succeed in mmap() the pages contained in the `pages` field of the `reg->gpu_alloc` to user space.

However, the strange situation can lead to memory corruption in a race condition:
```
Thread 1                                                                               Thread2
(processing the command `BASE_KCPU_COMMAND_TYPE_MAP_IMPORT`)

A1. enter the function `kbase_kcpu_map_import_prepare`:
`nents` field of `reg->gpu_alloc` gets initialized, while the `pages` field of 
`reg->gpu_alloc` is left uninitialized !!!

                                                                                       B1. mmap() the pages in the  `pages` field of  `reg->gpu_alloc` to user space with mmap() syscall. But the `pages` is not initialized yet!!!
                                                                                       By this operation, we just mmap() some physical pages which should never been accessed to the user space for accessing !!

A2. enter the function `kcpu_queue_process`:
`pages` field of `reg->gpu_alloc` gets initialized

```
