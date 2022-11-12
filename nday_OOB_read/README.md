##  The Basics
A already fixed issue I found when code auditing the old version of mali driver.

## The vulnerability
There is an ioctl command `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` in the ioctl of mali driver:
```
......
case KBASE_IOCTL_KCPU_QUEUE_ENQUEUE:
KBASE_HANDLE_IOCTL_IN(KBASE_IOCTL_KCPU_QUEUE_ENQUEUE,
kbasep_kcpu_queue_enqueue,
struct kbase_ioctl_kcpu_queue_enqueue,
kctx);
break;
.....
```
We can use the command `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` to enqueue a command into the KCPU queue. For example, we can use `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` to enqueue a queue command `BASE_KCPU_COMMAND_TYPE_GROUP_SUSPEND`.

The queue command `BASE_KCPU_COMMAND_TYPE_GROUP_SUSPEND` is processed with the function `kbase_csf_kcpu_queue_enqueue`. In the function `kbase_csf_kcpu_queue_enqueue`, the  queue command `BASE_KCPU_COMMAND_TYPE_GROUP_SUSPEND` will be processed first by function `kbase_csf_queue_group_suspend_prepare`, and then it will be processed by the function `kbase_csf_queue_group_suspend_process`.

In the function `kbase_csf_queue_group_suspend_process`, it calls the function `kbase_csf_queue_group_suspend`:
```
1int kbase_csf_queue_group_suspend(struct kbase_context *kctx,
2				  struct kbase_suspend_copy_buffer *sus_buf,
3				  u8 group_handle)
4{
5	struct kbase_device *const kbdev = kctx->kbdev;
6	int err;
7	struct kbase_queue_group *group;
8
9	err = kbase_reset_gpu_prevent_and_wait(kbdev);
10	if (err) {
11		dev_warn(
12			kbdev->dev,
13			"Unsuccessful GPU reset detected when suspending group %d",
14			group_handle);
15		return err;
16	}
17	mutex_lock(&kctx->csf.lock);
18
19	group = find_queue_group(kctx, group_handle);
20	if (group)
21		err = kbase_csf_scheduler_group_copy_suspend_buf(group,  //<-------- enter the routine !!!
22								 sus_buf);
23	else
24		err = -EINVAL;
25
26	mutex_unlock(&kctx->csf.lock);
27	kbase_reset_gpu_allow(kbdev);
28
29	return err;
30}
```
As you can see that the function `kbase_csf_scheduler_group_copy_suspend_buf` gets called:
```
1int kbase_csf_scheduler_group_copy_suspend_buf(struct kbase_queue_group *group,
2		struct kbase_suspend_copy_buffer *sus_buf)
3{
4	struct kbase_context *const kctx = group->kctx;
5	struct kbase_device *const kbdev = kctx->kbdev;
6	struct kbase_csf_scheduler *const scheduler = &kbdev->csf.scheduler;
7	int err = 0;
8
9	kbase_reset_gpu_assert_prevented(kbdev);
10	lockdep_assert_held(&kctx->csf.lock);
11	mutex_lock(&scheduler->lock);
12
13	if (kbasep_csf_scheduler_group_is_on_slot_locked(group)) {
14		DECLARE_BITMAP(slot_mask, MAX_SUPPORTED_CSGS) = {0};
15
16		set_bit(kbase_csf_scheduler_group_get_slot(group), slot_mask);
17
18		if (!WARN_ON(scheduler->state == SCHED_SUSPENDED))
19			suspend_queue_group(group);
20		err = wait_csg_slots_suspend(kbdev, slot_mask,
21					     kbdev->csf.fw_timeout_ms);
22		if (err) {
23			dev_warn(kbdev->dev, "[%llu] Timeout waiting for the group %d to suspend on slot %d",
24				 kbase_backend_get_cycle_cnt(kbdev),
25				 group->handle, group->csg_nr);
26			goto exit;
27		}
28	}
29
30	if (queue_group_suspended_locked(group)) {
31		unsigned int target_page_nr = 0, i = 0;
32		u64 offset = sus_buf->offset;
33		size_t to_copy = sus_buf->size;
34
35		if (scheduler->state != SCHED_SUSPENDED) {
36			/* Similar to the case of HW counters, need to flush
37			 * the GPU cache before reading from the suspend buffer
38			 * pages as they are mapped and cached on GPU side.
39			 */
40			kbase_gpu_start_cache_clean(kbdev);
41			kbase_gpu_wait_cache_clean(kbdev);
42		} else {
43			/* Make sure power down transitions have completed,
44			 * i.e. L2 has been powered off as that would ensure
45			 * its contents are flushed to memory.
46			 * This is needed as Scheduler doesn't wait for the
47			 * power down to finish.
48			 */
49			kbase_pm_wait_for_desired_state(kbdev);
50		}
51
52		for (i = 0; i < PFN_UP(sus_buf->size) &&
53				target_page_nr < sus_buf->nr_pages; i++) {
54			struct page *pg =
55				as_page(group->normal_suspend_buf.phy[i]);  //<--------- OOB read can happen here !!!!
56			void *sus_page = kmap(pg);
57
58			if (sus_page) {
59				kbase_sync_single_for_cpu(kbdev,
60					kbase_dma_addr(pg),
61					PAGE_SIZE, DMA_BIDIRECTIONAL);
62
63				err = kbase_mem_copy_to_pinned_user_pages(
64						sus_buf->pages, sus_page,
65						&to_copy, sus_buf->nr_pages,
66						&target_page_nr, offset);
67				kunmap(pg);
68				if (err)
69					break;
70			} else {
71				err = -ENOMEM;
72				break;
73			}
74		}
75		schedule_in_cycle(group, false);
76	} else {
77		/* If addr-space fault, the group may have been evicted */
78		err = -EIO;
79	}
80
81exit:
82	mutex_unlock(&scheduler->lock);
83	return err;
84}
```
As you can see, when performing the work of copying suspend buffer,  both `sus_buf->size` and `sus_buf->nr_pages` are used as the upper limit of the `for` circle. But  because there are not enough checks for both `sus_buf->size` and `sus_buf->nr_pages`, the `sus_buf->size` and `sus_buf->nr_pages` can be really big, as a result, the variable `i` used to traverse the array `group->normal_suspend_buf.phy` can be out of bound of the actual size of the array `group->normal_suspend_buf.phy`.  As  a result, the OOB read will happen and an illegal pointer of `struct page` will be got, information leak might happen for that.
