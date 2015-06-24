This page describes the rules processing subsystem

# Introduction #

For now this remains an internal help page


# Details #

high level has table for storing action structs per syscall (key=syscall nr., value=ptr to rule\_action list)

```
struct mem_val
  enum mode = {direct, one_reg, two_reg}
  int64 value //if mode = direct
  enum kvm_reg lhs //if mode = one_reg OR two_reg
  enum kvm_reg rhs //if mode = two_reg
  flag sign //if mode=two_reg, 1=plus 0=minus
```

```
struct rule_action
  u32 rule_id  //user-defined rule id
  flag halt //if 1, halt VM on rule hit
  flag direct //if 1, output value of target reg
  uint64 target_address //target memory address (direct may not equal 1)
  enum kvm_reg target_reg //if target_address=0, target register
  struct mem_val offset //if direct=0, indicates the offset to target
  struct mem_val size //if direct=0, indicates the size of the data to copy
  list_head siblings // list head to other rules for this syscall nr
```