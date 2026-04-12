# Addormentamento
```
      cyclictest-2641  [001]  2993.407426: funcgraph_entry:        0.875 us   |  cortex_a76_erratum_1463225_svc_handler();
      cyclictest-2641  [001]  2993.407428: funcgraph_entry:                   |  do_el0_svc() {
      cyclictest-2641  [001]  2993.407430: funcgraph_entry:                   |    el0_svc_common.constprop.0() {
      cyclictest-2641  [001]  2993.407432: funcgraph_entry:                   |      syscall_trace_enter() {
      cyclictest-2641  [001]  2993.407433: funcgraph_entry:                   |        __audit_syscall_entry() {
      cyclictest-2641  [001]  2993.407435: funcgraph_entry:        1.625 us   |          ktime_get_coarse_real_ts64();
      cyclictest-2641  [001]  2993.407438: funcgraph_exit:         5.000 us   |        }
      cyclictest-2641  [001]  2993.407440: funcgraph_exit:         8.125 us   |      }
      cyclictest-2641  [001]  2993.407441: funcgraph_entry:                   |      invoke_syscall() {
      cyclictest-2641  [001]  2993.407443: funcgraph_entry:                   |        __arm64_sys_clock_nanosleep() {
      cyclictest-2641  [001]  2993.407445: funcgraph_entry:        2.125 us   |          get_timespec64();
      cyclictest-2641  [001]  2993.407449: funcgraph_entry:                   |          common_nsleep() {
      cyclictest-2641  [001]  2993.407451: funcgraph_entry:                   |            hrtimer_nanosleep() {
      cyclictest-2641  [001]  2993.407453: funcgraph_entry:                   |              hrtimer_init_sleeper() {
      cyclictest-2641  [001]  2993.407455: funcgraph_entry:        1.875 us   |                __hrtimer_init();
      cyclictest-2641  [001]  2993.407458: funcgraph_exit:         5.125 us   |              }
      cyclictest-2641  [001]  2993.407460: funcgraph_entry:                   |              do_nanosleep() {
      cyclictest-2641  [001]  2993.407461: funcgraph_entry:                   |                hrtimer_start_range_ns() {
      cyclictest-2641  [001]  2993.407463: funcgraph_entry:                   |                  _raw_spin_lock_irqsave() {
      cyclictest-2641  [001]  2993.407465: funcgraph_entry:        0.875 us   |                    preempt_count_add();
      cyclictest-2641  [001]  2993.407466: funcgraph_exit:         3.625 us   |                  }
      cyclictest-2641  [001]  2993.407467: funcgraph_entry:                   |                  ktime_get() {
      cyclictest-2641  [001]  2993.407468: funcgraph_entry:        0.750 us   |                    arch_counter_read();
      cyclictest-2641  [001]  2993.407470: funcgraph_exit:         2.500 us   |                  }
      cyclictest-2641  [001]  2993.407471: funcgraph_entry:                   |                  get_nohz_timer_target() {
      cyclictest-2641  [001]  2993.407472: funcgraph_entry:        0.750 us   |                    housekeeping_test_cpu();
      cyclictest-2641  [001]  2993.407473: funcgraph_exit:         2.500 us   |                  }
      cyclictest-2641  [001]  2993.407475: funcgraph_entry:        1.125 us   |                  enqueue_hrtimer();
      cyclictest-2641  [001]  2993.407477: funcgraph_entry:        0.875 us   |                  hrtimer_reprogram.constprop.0();
      cyclictest-2641  [001]  2993.407478: funcgraph_entry:                   |                  _raw_spin_unlock_irqrestore() {
      cyclictest-2641  [001]  2993.407480: funcgraph_entry:        1.750 us   |                    preempt_count_sub();
      cyclictest-2641  [001]  2993.407483: funcgraph_exit:         4.500 us   |                  }
      cyclictest-2641  [001]  2993.407485: funcgraph_exit:       + 23.375 us  |                }
      cyclictest-2641  [001]  2993.407486: funcgraph_entry:                   |                schedule() {
      cyclictest-2641  [001]  2993.407488: funcgraph_entry:        1.625 us   |                  preempt_count_add();
      cyclictest-2641  [001]  2993.407491: funcgraph_entry:                   |                  rcu_note_context_switch() {
      cyclictest-2641  [001]  2993.407492: funcgraph_entry:        0.875 us   |                    rcu_qs();
      cyclictest-2641  [001]  2993.407494: funcgraph_exit:         2.625 us   |                  }
      cyclictest-2641  [001]  2993.407494: funcgraph_entry:        0.875 us   |                  preempt_count_add();
      cyclictest-2641  [001]  2993.407496: funcgraph_entry:                   |                  _raw_spin_lock() {
      cyclictest-2641  [001]  2993.407497: funcgraph_entry:        0.750 us   |                    preempt_count_add();
      cyclictest-2641  [001]  2993.407498: funcgraph_exit:         2.500 us   |                  }
      cyclictest-2641  [001]  2993.407499: funcgraph_entry:        0.875 us   |                  preempt_count_sub();
      cyclictest-2641  [001]  2993.407501: funcgraph_entry:                   |                  update_rq_clock.part.0() {
      cyclictest-2641  [001]  2993.407502: funcgraph_entry:        0.875 us   |                    update_irq_load_avg();
      cyclictest-2641  [001]  2993.407503: funcgraph_exit:         2.625 us   |                  }
      cyclictest-2641  [001]  2993.407504: funcgraph_entry:                   |                  dequeue_task_fair() {
      cyclictest-2641  [001]  2993.407505: funcgraph_entry:                   |                    __update_curr.isra.0() {
      cyclictest-2641  [001]  2993.407506: funcgraph_entry:        0.875 us   |                      update_min_vruntime();
      cyclictest-2641  [001]  2993.407508: funcgraph_entry:                   |                      __cgroup_account_cputime() {
      cyclictest-2641  [001]  2993.407509: funcgraph_entry:        0.750 us   |                        preempt_count_add();
      cyclictest-2641  [001]  2993.407511: funcgraph_entry:        1.000 us   |                        cgroup_rstat_updated();
      cyclictest-2641  [001]  2993.407512: funcgraph_entry:        0.875 us   |                        preempt_count_sub();
      cyclictest-2641  [001]  2993.407514: funcgraph_exit:         6.000 us   |                      }
      cyclictest-2641  [001]  2993.407515: funcgraph_exit:         9.875 us   |                    }
      cyclictest-2641  [001]  2993.407516: funcgraph_entry:        0.875 us   |                    __update_load_avg_se();
      cyclictest-2641  [001]  2993.407517: funcgraph_entry:        0.875 us   |                    __update_load_avg_cfs_rq();
      cyclictest-2641  [001]  2993.407519: funcgraph_entry:        1.000 us   |                    avg_vruntime();
      cyclictest-2641  [001]  2993.407521: funcgraph_entry:        1.000 us   |                    update_min_vruntime();
      cyclictest-2641  [001]  2993.407523: funcgraph_exit:       + 18.500 us  |                  }
      cyclictest-2641  [001]  2993.407523: funcgraph_entry:                   |                  pick_next_task_fair() {
      cyclictest-2641  [001]  2993.407524: funcgraph_entry:        0.875 us   |                    put_prev_task_fair();
      cyclictest-2641  [001]  2993.407526: funcgraph_entry:        0.875 us   |                    __pick_eevdf();
      cyclictest-2641  [001]  2993.407528: funcgraph_entry:                   |                    set_next_entity() {
      cyclictest-2641  [001]  2993.407528: funcgraph_entry:        1.000 us   |                      __dequeue_entity();
      cyclictest-2641  [001]  2993.407530: funcgraph_entry:        0.875 us   |                      __update_load_avg_se();
      cyclictest-2641  [001]  2993.407532: funcgraph_entry:        0.875 us   |                      __update_load_avg_cfs_rq();
      cyclictest-2641  [001]  2993.407533: funcgraph_exit:         6.000 us   |                    }
      cyclictest-2641  [001]  2993.407534: funcgraph_exit:       + 10.875 us  |                  }
      cyclictest-2641  [001]  2993.407536: funcgraph_entry:                   |                  check_and_switch_context() {
      cyclictest-2641  [001]  2993.407537: funcgraph_entry:                   |                    cpu_do_switch_mm() {
      cyclictest-2641  [001]  2993.407538: funcgraph_entry:        0.875 us   |                      post_ttbr_update_workaround();
      cyclictest-2641  [001]  2993.407539: funcgraph_exit:         2.375 us   |                    }
      cyclictest-2641  [001]  2993.407540: funcgraph_exit:         4.125 us   |                  }
```


