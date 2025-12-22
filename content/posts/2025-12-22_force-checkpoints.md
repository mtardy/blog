---
title: "Force checkpoints"
date: "2025-12-22"
categories: ["ebpf"]
tags: ["ebpf", "linux", "C"]
slug: "force-checkpoints"
---

*This article is part of a series of notes that Paul Chaignon and I wrote to prepare a
presentation introducing the BPF verifier state pruning for Linux Plumbers 2025
in Tokyo, see [Making Sense of State Pruning](https://lpc.events/event/19/contributions/2162/).
You can also find the [slides](https://lpc.events/event/19/contributions/2162/attachments/1820/3904/LPC25_State_Pruning.pdf)
and the [video recording](http://www.youtube.com/watch?v=EoEBkFJ3St4) of the
presentation.*

*The version of the kernel code for these notes is `v6.17`.*

## Force checkpoints

### History

- This patch from March 2023 introduced `force_checkpoint` [4b5ce570dbef ("bpf:
  ensure state checkpointing at iter_next() call
  sites")](https://lore.kernel.org/all/20230310060149.625887-1-andrii@kernel.org/).
- Then another patch in October 2023 used `is_force_checkpoint` again
  [2793a8b015f7 ("bpf: exact states comparison for iterator convergence
  checks")](https://lore.kernel.org/all/20231024000917.12153-4-eddyz87@gmail.com/).
- Then a new patch in November 2023 added a new case in which we call
  `mark_force_checkpoint` [ab5cfac139ab ("bpf: verify callbacks as if they are
  called unknown number of times")](https://lore.kernel.org/r/20231121020701.26440-7-eddyz87@gmail.com).
- Finally, another one in March 2024 added another case to call
  `mark_force_checkpoint` [011832b97b31 ("bpf: Introduce may_goto
  instruction")](https://lore.kernel.org/bpf/20240306031929.42666-2-alexei.starovoitov@gmail.com).

### Mark force checkpoints

Force checkpoints, similarly to pruning points, are set in `visit_isns`. In all
three cases visited below, the force checkpoint is set on the instruction
itself.

#### Sync callbacks

The first call to `mark_force_checkpoint` is when handling `BPF_CALL` opcode
with the `is_sync_callback_calling_insn`.

```c
	case BPF_CALL:
        [...]
		/* For functions that invoke callbacks it is not known how many times
		 * callback would be called. Verifier models callback calling functions
		 * by repeatedly visiting callback bodies and returning to origin call
		 * instruction.
		 * In order to stop such iteration verifier needs to identify when a
		 * state identical some state from a previous iteration is reached.
		 * Check below forces creation of checkpoint before callback calling
		 * instruction to allow search for such identical states.
		 */
		if (is_sync_callback_calling_insn(insn)) {
			mark_calls_callback(env, t);
			mark_force_checkpoint(env, t);
			mark_prune_point(env, t);
			mark_jmp_point(env, t);
		}
```

#### Open coded iterators

The second case is also in `BPF_CALL`, in the call to kfunc and
`is_iter_next_kfunc` case.

```c
		} else if (insn->src_reg == BPF_PSEUDO_KFUNC_CALL) {
			struct bpf_kfunc_call_arg_meta meta;

			ret = fetch_kfunc_meta(env, insn, &meta, NULL);
			if (ret == 0 && is_iter_next_kfunc(&meta)) {
				mark_prune_point(env, t);
				/* Checking and saving state checkpoints at iter_next() call
				 * is crucial for fast convergence of open-coded iterator loop
				 * logic, so we need to force it. If we don't do that,
				 * is_state_visited() might skip saving a checkpoint, causing
				 * unnecessarily long sequence of not checkpointed
				 * instructions and jumps, leading to exhaustion of jump
				 * history buffer, and potentially other undesired outcomes.
				 * It is expected that with correct open-coded iterators
				 * convergence will happen quickly, so we don't run a risk of
				 * exhausting memory.
				 */
				mark_force_checkpoint(env, t);
			}
```

#### May goto instruction

The last call to `mark_force_checkpoint` is in the `default` case, meaning for
conditional jumps, in the specific case of `is_may_goto_insn`.

```c
	default:
		/* conditional jump with two edges */
		mark_prune_point(env, t);
		if (is_may_goto_insn(insn))
			mark_force_checkpoint(env, t);
```

The `may_goto` instruction is special and relatively new, from March 2024,
added by [011832b97b31 ("bpf: Introduce may_goto
instruction")](https://lore.kernel.org/bpf/20240306031929.42666-2-alexei.starovoitov@gmail.com).
It uses the `BPF_JCOND` opcode `0xe0` with the source register being
`BPF_MAY_GOTO = 0`. Note that this instruction is not documented in the
[instruction set kernel documentation for v6.17](https://www.kernel.org/doc/html/v6.17/bpf/standardization/instruction-set.html#jump-instructions).

```c
static bool is_may_goto_insn(struct bpf_insn *insn)
{
	return insn->code == (BPF_JMP | BPF_JCOND) && insn->src_reg == BPF_MAY_GOTO;
}
```

On a side note, from the commit description, the same instruction opcode would
be reused with source register being `goto_or_nop`, I imagine for BPF static
keys [from Anton Protopopov](https://lwn.net/Articles/1017439/).

As I understand, the goal of this instruction is to be injected in loops (also
potentially by LLVM) to offload the verifier responsability of loop
verification to a runtime check. The runtime check being decrementing a counter
so far.

#### Conclusion

In all the above cases, it seems the force checkpoints are tied to looping
mechanisms:
- Sync callback are typically `bpf_for_each_map_elem`, `bpf_find_vma`,
  `bpf_loop`, `bpf_user_ringbuf_drain`, etc.
- Open coded iterators are typically used via the `bpf_for` or `bpf_repeat`
  macro.
- The may goto instruction is tied to the offload of the verification of loops
  to the BPF runtime.

### Reading force checkpoints

From the initial commit adding force checkpoints ([4b5ce570dbef ("bpf: ensure
state checkpointing at iter_next() call
sites")](https://lore.kernel.org/all/20230310060149.625887-1-andrii@kernel.org/)),
the only place that force checkpoints were checked was (and actually still is)
`is_state_visited`, which is the function that is called in the main verifier
loop `do_check` if the current instruction is a prune point.

This means that the force checkpoint is a mechanism that adds on top of the
pruning points.

#### Force the creation of a new state

So the first use of this is to force the creation a new state, as we could have
imagined.

```diff
@@ -15172,7 +15196,8 @@ static int is_state_visited(struct bpf_verifier_env *env, int insn_idx)
 	struct bpf_verifier_state_list *sl, **pprev;
 	struct bpf_verifier_state *cur = env->cur_state, *new;
 	int i, j, err, states_cnt = 0;
-	bool add_new_state = env->test_state_freq ? true : false;
+	bool force_new_state = env->test_state_freq || is_force_checkpoint(env, insn_idx);
+	bool add_new_state = force_new_state;
```

We saw that force checkpoints are tied to looping mechanisms, highlighting
this, this specific line was updated in October 2024 by [aa30eb3260b2 ("bpf:
Force checkpoint when jmp history is too
long")](https://lore.kernel.org/bpf/20241029172641.1042523-1-eddyz87@gmail.com/).
The commit was fixing situations in which the verifier could be extremely slow
trying to verify programs with particular loops, see the commit description for
more info.

```c
	force_new_state = env->test_state_freq || is_force_checkpoint(env, insn_idx) ||
			  /* Avoid accumulating infinitely long jmp history */
			  cur->jmp_history_cnt > 40;
```

#### Increase the number of miss before state eviction

The other use of `is_force_checkpoint` is less obvious, it was added by
[2793a8b015f7 ("bpf: exact states comparison for iterator convergence
checks")](https://lore.kernel.org/all/20231024000917.12153-4-eddyz87@gmail.com/).

At the end of `is_state_visited` lies a heuristic to figure out if we should
keep the state or not for checking equivalence. On one side, finding state
equivalence reduces the number of instructions processed, thus complexity of
programs, but on the other side, keeping a large number of state increases the
verification time, so we need to find a balance.

In this heuristic, when the instruction is a force checkpoint, the tolerance
for state miss is significantly higher:
> Use bigger 'n' for checkpoints because evicting checkpoint states too early
> would hinder iterator convergence.

```c
		/* heuristic to determine whether this state is beneficial
		 * to keep checking from state equivalence point of view.
		 * Higher numbers increase max_states_per_insn and verification time,
		 * but do not meaningfully decrease insn_processed.
		 * 'n' controls how many times state could miss before eviction.
		 * Use bigger 'n' for checkpoints because evicting checkpoint states
		 * too early would hinder iterator convergence.
		 */
		n = is_force_checkpoint(env, insn_idx) && sl->state.branches > 0 ? 64 : 3;
		if (sl->miss_cnt > sl->hit_cnt * n + n) {
			/* the state is unlikely to be useful. Remove it to
			 * speed up verification
			 */
			sl->in_free_list = true;
			list_del(&sl->node);
			list_add(&sl->node, &env->free_list);
			env->free_list_size++;
			env->explored_states_size--;
			maybe_free_verifier_state(env, sl);
		}
```
