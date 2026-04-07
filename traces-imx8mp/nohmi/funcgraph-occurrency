      cyclictest-1905  [001]  1621.873548: funcgraph_entry:        0.875 us   |  cortex_a76_erratum_1463225_svc_handler();
      cyclictest-1905  [001]  1621.873550: funcgraph_entry:                   |  do_el0_svc() {
      cyclictest-1905  [001]  1621.873552: funcgraph_entry:                   |    el0_svc_common.constprop.0() {
      cyclictest-1905  [001]  1621.873555: funcgraph_entry:                   |      syscall_trace_enter() {
      cyclictest-1905  [001]  1621.873557: funcgraph_entry:        3.625 us   |        __audit_syscall_entry();
      cyclictest-1905  [001]  1621.873562: funcgraph_exit:         6.750 us   |      }
      cyclictest-1905  [001]  1621.873564: funcgraph_entry:                   |      invoke_syscall() {
      cyclictest-1905  [001]  1621.873565: funcgraph_entry:                   |        __arm64_sys_clock_nanosleep() {
       ktimers/1-27    [001]  1621.883717: funcgraph_entry:                   |  _raw_spin_lock_irqsave() {
       ktimers/1-27    [001]  1621.883718: funcgraph_entry:        1.000 us   |    preempt_count_add();
       ktimers/1-27    [001]  1621.883722: funcgraph_exit:         4.750 us   |  }
       ktimers/1-27    [001]  1621.883722: funcgraph_entry:                   |  _raw_spin_unlock_irqrestore() {
       ktimers/1-27    [001]  1621.883723: funcgraph_entry:        0.875 us   |    preempt_count_sub();
       ktimers/1-27    [001]  1621.883725: funcgraph_exit:         2.375 us   |  }
       ktimers/1-27    [001]  1621.883726: funcgraph_entry:                   |  fpsimd_thread_switch() {
       ktimers/1-27    [001]  1621.883727: funcgraph_entry:        1.625 us   |    __get_cpu_fpsimd_context();
       ktimers/1-27    [001]  1621.883729: funcgraph_entry:        0.875 us   |    fpsimd_save();
       ktimers/1-27    [001]  1621.883731: funcgraph_entry:        0.875 us   |    __put_cpu_fpsimd_context();
       ktimers/1-27    [001]  1621.883732: funcgraph_exit:         6.625 us   |  }
       ktimers/1-27    [001]  1621.883733: funcgraph_entry:        0.875 us   |  tls_preserve_current_state();
       ktimers/1-27    [001]  1621.883735: funcgraph_entry:        0.875 us   |  hw_breakpoint_thread_switch();
       ktimers/1-27    [001]  1621.883736: funcgraph_entry:                   |  spectre_v4_enable_task_mitigation() {
       ktimers/1-27    [001]  1621.883737: funcgraph_entry:        0.875 us   |    cpu_mitigations_off();
       ktimers/1-27    [001]  1621.883739: funcgraph_entry:        0.875 us   |    cpu_mitigations_off();
       ktimers/1-27    [001]  1621.883740: funcgraph_exit:         4.125 us   |  }
       ktimers/1-27    [001]  1621.883741: funcgraph_entry:                   |  erratum_1418040_thread_switch() {
       ktimers/1-27    [001]  1621.883742: funcgraph_entry:                   |    this_cpu_has_cap() {
       ktimers/1-27    [001]  1621.883743: funcgraph_entry:        0.875 us   |      is_affected_midr_range_list();
       ktimers/1-27    [001]  1621.883745: funcgraph_exit:         2.500 us   |    }
       ktimers/1-27    [001]  1621.883745: funcgraph_exit:         4.000 us   |  }
       ktimers/1-27    [001]  1621.883746: funcgraph_entry:        0.875 us   |  mte_thread_switch();
      cyclictest-1905  [001]  1621.883750: funcgraph_exit:       # 10184.750 us |        }
      cyclictest-1905  [001]  1621.883753: funcgraph_exit:       # 10189.625 us |      }
      cyclictest-1905  [001]  1621.883755: funcgraph_entry:                   |      syscall_trace_exit() {
      cyclictest-1905  [001]  1621.883756: funcgraph_entry:        1.625 us   |        __audit_syscall_exit();
      cyclictest-1905  [001]  1621.883761: funcgraph_exit:         6.250 us   |      }
      cyclictest-1905  [001]  1621.883762: funcgraph_exit:       # 10210.375 us |    }
      cyclictest-1905  [001]  1621.883764: funcgraph_exit:       # 10213.500 us |  }
      cyclictest-1905  [001]  1621.883767: funcgraph_entry:                   |  do_notify_resume() {
      cyclictest-1905  [001]  1621.883769: funcgraph_entry:        3.250 us   |    blkcg_maybe_throttle_current();
      cyclictest-1905  [001]  1621.883773: funcgraph_entry:                   |    __rseq_handle_notify_resume() {
      cyclictest-1905  [001]  1621.883777: funcgraph_entry:        1.500 us   |      clear_rseq_cs.isra.0();
      cyclictest-1905  [001]  1621.883781: funcgraph_exit:         8.125 us   |    }
      cyclictest-1905  [001]  1621.883783: funcgraph_entry:                   |    fpsimd_restore_current_state() {
      cyclictest-1905  [001]  1621.883785: funcgraph_entry:                   |      get_cpu_fpsimd_context() {
      cyclictest-1905  [001]  1621.883786: funcgraph_entry:        3.250 us   |        preempt_count_add();
      cyclictest-1905  [001]  1621.883791: funcgraph_exit:         6.250 us   |      }
      cyclictest-1905  [001]  1621.883794: funcgraph_entry:        1.625 us   |      task_fpsimd_load();
      cyclictest-1905  [001]  1621.883799: funcgraph_entry:        1.625 us   |      fpsimd_bind_task_to_cpu();
      cyclictest-1905  [001]  1621.883802: funcgraph_entry:                   |      put_cpu_fpsimd_context() {
      cyclictest-1905  [001]  1621.883805: funcgraph_entry:        1.625 us   |        preempt_count_sub();
      cyclictest-1905  [001]  1621.883810: funcgraph_exit:         7.875 us   |      }
      cyclictest-1905  [001]  1621.883811: funcgraph_exit:       + 28.125 us  |    }
      cyclictest-1905  [001]  1621.883814: funcgraph_exit:       + 47.250 us  |  }
