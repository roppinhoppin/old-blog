---
layout: post
title: "What does kAFL patch to linux kernel"
---

**arch/x86/kvm/vmx_pt.c**
**arch/x86/kvm/vmx_pt.h**
The *core* of KVM-PT, implements:
- 

**arch/x86/include/asm/kvm_host.h.patch**
```
> #ifdef CONFIG_KVM_VMX_PT
> 	int (*setup_trace_fd)(struct kvm_vcpu *vcpu);
> 	int (*vmx_pt_enabled)(void);
> #endif
```

**arch/x86/include/uapi/asm/kvm.h.patch**
```
> /* vmx_pt */
> struct vmx_pt_filter_iprs {
> 	__u64 a;
> 	__u64 b;
> };
> 
```
Configures the filter value from a and b.

**arch/x86/kvm/svm.c.patch**
```
> #ifdef CONFIG_KVM_VMX_PT
> static int setup_trace_fd_stub(struct kvm_vcpu *vcpu){
> 	return -EINVAL;
> }
> static int vmx_pt_is_enabled(void){
> 	/* AMD CPUs do not support Intel PT */
> 	return -EINVAL;
> }
> #endif	
> 
4417a4428,4432
> 	
> #ifdef CONFIG_KVM_VMX_PT
> 	.setup_trace_fd = setup_trace_fd_stub,
> 	.vmx_pt_enabled = vmx_pt_is_enabled,
> #endif	
```
Since AMD doesn't support the PT feature, this patch disables for them.

**arch/x86/kvm/vmx.c.patch**
```
54a55,59
> #ifdef CONFIG_KVM_VMX_PT
> #include "vmx_pt.h"
> static int handle_monitor_trap(struct kvm_vcpu *vcpu);
> #endif
> 
521a527,529
> #ifdef CONFIG_KVM_VMX_PT
> 	struct vcpu_vmx_pt*   vmx_pt_config;
> #endif
1608a1617,1619
> #ifdef CONFIG_KVM_VMX_PT
> 	vmx->vm_entry_controls_shadow = val | 0x20000ULL;	/* Conceal VM entries from Intel PT */
> #else
1609a1621
> #endif
1636a1649,1651
> #ifdef CONFIG_KVM_VMX_PT
> 	vmx->vm_exit_controls_shadow = val | 0x1000000ULL;	/* Conceal VM exit from Intel PT */
> #else
1637a1653
> #endif
1801c1817
< static void add_atomic_switch_msr(struct vcpu_vmx *vmx, unsigned msr,
---
> void add_atomic_switch_msr(struct vcpu_vmx *vmx, unsigned msr,
3346c3362
< 	vmcs_conf->cpu_based_exec_ctrl = _cpu_based_exec_control;
---
> 	vmcs_conf->cpu_based_exec_ctrl = _cpu_based_exec_control | 0x80000;
3348,3349c3364,3365
< 	vmcs_conf->vmexit_ctrl         = _vmexit_control;
< 	vmcs_conf->vmentry_ctrl        = _vmentry_control;
---
> 	vmcs_conf->vmexit_ctrl         = _vmexit_control | 0x1000000;
> 	vmcs_conf->vmentry_ctrl        = _vmentry_control | 0x20000;
4809a4826
> 				
5445d5461
< 
8245a8262
> 
8618a8636,8639
> #ifdef CONFIG_KVM_VMX_PT
> 	vmx_pt_vmentry(vmx->vmx_pt_config);
> #endif
> 
8825a8847,8850
> 	
> 	#ifdef CONFIG_KVM_VMX_PT
> 		vmx_pt_vmexit(vmx->vmx_pt_config);
> 	#endif
8855a8881,8885
> 	
> #ifdef CONFIG_KVM_VMX_PT
> 	/* free vmx_pt */
> 	vmx_pt_destroy(vmx, &(vmx->vmx_pt_config));
> #endif
8937a8968,8972
> #ifdef CONFIG_KVM_VMX_PT
> 	/* enable vmx_pt */
> 	vmx_pt_setup(vmx, &(vmx->vmx_pt_config));
> #endif
> 
10891a10927,10936
> #ifdef CONFIG_KVM_VMX_PT
> static int vmx_pt_setup_fd(struct kvm_vcpu *vcpu){
> 	return vmx_pt_create_fd(to_vmx(vcpu)->vmx_pt_config);
> }
> 
> static int vmx_pt_is_enabled(void){
> 	return vmx_pt_enabled();
> }
> #endif	
> 
11015a11061,11065
> 	
> #ifdef CONFIG_KVM_VMX_PT
> 	.setup_trace_fd = vmx_pt_setup_fd,
> 	.vmx_pt_enabled = vmx_pt_is_enabled,
> #endif	
11029a11080,11082
> #ifdef CONFIG_KVM_VMX_PT
> 	vmx_pt_init();
> #endif
11039c11092,11094
< 
---
> #ifdef CONFIG_KVM_VMX_PT
> 	vmx_pt_exit();
> #endif
```
This patch implements the interface between kvm and userspace

**arch/x86/kvm/vmx.h**
```
#ifndef __VMX_H__
#define __VMX_H__

#include <linux/types.h>
#include <linux/uaccess.h>

struct vcpu_vmx;
void add_atomic_switch_msr(struct vcpu_vmx *vmx, unsigned msr, u64 guest_val, u64 host_val);

#endif
```

**arch/x86/kvm/x86.c.patch**
```
> #ifdef CONFIG_KVM_VMX_PT
> 	case KVM_VMX_PT_SUPPORTED: {
> 		r = kvm_x86_ops->vmx_pt_enabled();
> 		break;
> 	}
> #endif
3498a3505,3510
> #ifdef CONFIG_KVM_VMX_PT
> 	case KVM_VMX_PT_SETUP_FD: {
> 		r = kvm_x86_ops->setup_trace_fd(vcpu);
> 		break;
> 	}
> #endif
5970a5983,6003
> #ifdef CONFIG_KVM_VMX_PT
> 	/* kAFL Hypercall Interface (ring 0) */
> 	if(kvm_x86_ops->get_cpl(vcpu) == 0) {
> 		r = 0;
> 		if (kvm_register_read(vcpu, VCPU_REGS_RAX) == HYPERCALL_KAFL_RAX_ID){
> 			switch(kvm_register_read(vcpu, VCPU_REGS_RBX)){
> 				case 8: /* PANIC */   
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_PANIC;    
> 					break;
> 				case 9: /* KASAN */ 
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_KASAN; 
> 					break;
> 				default:
> 					r = -KVM_EPERM;   
> 					break;   
> 			}
> 			return r;
> 		}
> 	}
> #endif
> 
5971a6005,6061
> 		/* kAFL Hypercall interface */
> 		#ifdef CONFIG_KVM_VMX_PT
> 		if (kvm_register_read(vcpu, VCPU_REGS_RAX) == HYPERCALL_KAFL_RAX_ID){
> 			r = 0;
> 			switch(kvm_register_read(vcpu, VCPU_REGS_RBX)){
> 				case 0:  /* KAFL_GUEST_ACQUIRE */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_ACQUIRE;
> 					break;
> 				case 1:  /* KAFL_GUEST_GET_PAYLOAD */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_GET_PAYLOAD;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX);
> 					break;
> 				case 2:  /* KAFL_GUEST_GET_PROGRAM */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_GET_PROGRAM;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX);
> 					break;
> 				case 3: /* KAFL_GUEST_GET_ARGV */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_GET_ARGV;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX);
> 					break;
> 				case 4: /* KAFL_GUEST_RELEASE */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_RELEASE;
> 					break;
> 				case 5: /* KAFL_GUEST_SUBMIT_CR3 */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_SUBMIT_CR3;
> 					vcpu->run->hypercall.args[0] = kvm_read_cr3(vcpu);
> 					break;
> 				case 6: /* KAFL_GUEST_PANIC */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_SUBMIT_PANIC;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX); 
> 					break;
> 				case 7: /* KAFL_GUEST_KASAN */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_SUBMIT_KASAN;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX); 
> 					break;
> 				case 10: /* LOCK */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_LOCK;
> 					break;
> 				case 11: /* INFO */    
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_INFO;
>                     vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX);
> 					break;
> 				case 12: /* KAFL_GUEST_NEXT_PAYLOAD */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_NEXT_PAYLOAD;
> 					break;
> 				case 13: /* KAFL_GUEST_DEBUG */
> 					vcpu->run->exit_reason = KVM_EXIT_KAFL_DEBUG;
> 					vcpu->run->hypercall.args[0] = kvm_register_read(vcpu, VCPU_REGS_RCX);
> 					break;
> 				default:
> 					r = -KVM_EPERM;
> 					break;
> 			}
> 			return r;
> 		}
> 		#endif
> 
```

**include/uapi/linux/kvm.h.patch**
```
> #define HYPERCALL_KAFL_RAX_ID			0x01f
> #define KVM_EXIT_KAFL_ACQUIRE			100
> #define KVM_EXIT_KAFL_GET_PAYLOAD		101
> #define KVM_EXIT_KAFL_GET_PROGRAM		102
> #define KVM_EXIT_KAFL_GET_ARGV		103
> #define KVM_EXIT_KAFL_RELEASE			104
> #define KVM_EXIT_KAFL_SUBMIT_CR3		105
> #define KVM_EXIT_KAFL_SUBMIT_PANIC	106
> #define KVM_EXIT_KAFL_SUBMIT_KASAN	107
> #define KVM_EXIT_KAFL_PANIC			108
> #define KVM_EXIT_KAFL_KASAN			109
> #define KVM_EXIT_KAFL_LOCK			110
> #define KVM_EXIT_KAFL_INFO			111
> #define KVM_EXIT_KAFL_NEXT_PAYLOAD	112
> #define KVM_EXIT_KAFL_DEBUG			113
> 
1313a1330,1359
> 
> /*
>  * ioctls for vmx_pt fds
>  */
> #define KVM_VMX_PT_SETUP_FD				_IO(KVMIO,	0xd0)			/* apply vmx_pt fd (via vcpu fd ioctl)*/
> #define KVM_VMX_PT_CONFIGURE_ADDR0		_IOW(KVMIO,	0xd1, __u64)	/* configure IP-filtering for addr0_a & addr0_b */
> #define KVM_VMX_PT_CONFIGURE_ADDR1		_IOW(KVMIO,	0xd2, __u64)	/* configure IP-filtering for addr1_a & addr1_b */
> #define KVM_VMX_PT_CONFIGURE_ADDR2		_IOW(KVMIO,	0xd3, __u64)	/* configure IP-filtering for addr2_a & addr2_b */
> #define KVM_VMX_PT_CONFIGURE_ADDR3		_IOW(KVMIO,	0xd4, __u64)	/* configure IP-filtering for addr3_a & addr3_b */
> 
> #define KVM_VMX_PT_CONFIGURE_CR3			_IOW(KVMIO,	0xd5, __u64)	/* setup CR3 filtering value */
> #define KVM_VMX_PT_ENABLE					_IO(KVMIO,	0xd6)			/* enable and lock configuration */ 
> #define KVM_VMX_PT_GET_TOPA_SIZE			_IOR(KVMIO,	0xd7, __u32)	/* get defined ToPA size */
> #define KVM_VMX_PT_DISABLE				_IO(KVMIO,	0xd8)			/* enable and lock configuration */ 
> #define KVM_VMX_PT_CHECK_TOPA_OVERFLOW	_IO(KVMIO,	0xd9)			/* check for ToPA overflow */
> 
> #define KVM_VMX_PT_ENABLE_ADDR0			_IO(KVMIO,	0xaa)			/* enable IP-filtering for addr0 */
> #define KVM_VMX_PT_ENABLE_ADDR1			_IO(KVMIO,	0xab)			/* enable IP-filtering for addr1 */
> #define KVM_VMX_PT_ENABLE_ADDR2			_IO(KVMIO,	0xac)			/* enable IP-filtering for addr2 */
> #define KVM_VMX_PT_ENABLE_ADDR3			_IO(KVMIO,	0xad)			/* enable IP-filtering for addr3 */
> 
> #define KVM_VMX_PT_DISABLE_ADDR0			_IO(KVMIO,	0xae)			/* disable IP-filtering for addr0 */
> #define KVM_VMX_PT_DISABLE_ADDR1			_IO(KVMIO,	0xaf)			/* disable IP-filtering for addr1 */
> #define KVM_VMX_PT_DISABLE_ADDR2			_IO(KVMIO,	0xe0)			/* disable IP-filtering for addr2 */
> #define KVM_VMX_PT_DISABLE_ADDR3			_IO(KVMIO,	0xe1)			/* disable IP-filtering for addr3 */
> 
> #define KVM_VMX_PT_ENABLE_CR3				_IO(KVMIO,	0xe2)			/* enable CR3 filtering */
> #define KVM_VMX_PT_DISABLE_CR3			_IO(KVMIO,	0xe3)			/* disable CR3 filtering */
> 
> #define KVM_VMX_PT_SUPPORTED				_IO(KVMIO,	0xe4)
```