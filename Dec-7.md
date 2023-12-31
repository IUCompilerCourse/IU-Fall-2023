# Review for Final Exam

Overview of the topics on the final exam:

* Tuples and Garbage Collection
* Functions

# Tuples and Garbage Collection

New features:
* tuples

Main issues:

* tuples live forever, so we put them on the heap

* but unused tuples should go away, so we perform garbage collection

## Source Program

	(let ([a (vector 777)])
	   (let ([b (vector a a)])
		  (let ([_ (vector-set! (vector-ref b 0) 0 42)])
			 (vector-ref a 0))))

## Expose Allocation

Compile `(vector ...)` into possible call to `collect` followed by `allocate` operation.

	(let ([a57 (let ([_62 (if (< (+ (global-value free_ptr) 16) (global-value fromspace_end))
								   (void)
								   (collect 16))
								])
					 (let ([alloc60 (allocate 1 (Vector Integer))])
						(let ([_61 (vector-set! alloc60 0 777)])
						   alloc60)))])
	   (let ([b58 (let ([_66 (if (< (+ (global-value free_ptr) 24) (global-value fromspace_end))
									  (void)
									  (collect 24))
								   ])
						(let ([alloc63 (allocate 2 (Vector (Vector Integer) (Vector Integer)))])
						   (let ([_65 (vector-set! alloc63 0 a57)])
							  (let ([_64 (vector-set! alloc63 1 a57)])
								 alloc63))))])
		  (let ([_59 (vector-set! (vector-ref b58 0) 0 42)])
			 (vector-ref a57 0))))

## Explicate Control

(nothing interesting here, skip to select instructions)

	block79:
		(collect 16)
		goto block77;
	block78:
		_62 = (void);
		goto block77;
	block77:
		alloc60 = (allocate 1 (Vector Integer));
		_61 = (vector-set! alloc60 0 777);
		a57 = alloc60;
		tmp70 = (global-value free_ptr);
		tmp71 = (+ tmp70 24);
		tmp72 = (global-value fromspace_end);
		if (< tmp71 tmp72)
		   goto block75;
		else
		   goto block76;
	block76:
		(collect 24)
		goto block74;
	block75:
		_66 = (void);
		goto block74;
	block74:
		alloc63 = (allocate 2 (Vector (Vector Integer) (Vector Integer)));
		_65 = (vector-set! alloc63 0 a57);
		_64 = (vector-set! alloc63 1 a57);
		b58 = alloc63;
		tmp73 = (vector-ref b58 0);
		_59 = (vector-set! tmp73 0 42);
		return (vector-ref a57 0);
	start:
		tmp67 = (global-value free_ptr);
		tmp68 = (+ tmp67 16);
		tmp69 = (global-value fromspace_end);
		if (< tmp68 tmp69)
		   goto block78;
		else
		   goto block79;


## Select Instructions

Compile `vector-ref` and `vector-set!` into `movq` instructions.

Compile `allocate` into a bump of the free pointer and setting the tag.

Compile `collect` into a call to `collect`.

Compile global variables into PC-relative addressing.

    
	block79:
		movq %r15, %rdi
		movq $16, %rsi
		callq collect
		jmp block77

	block78:
		movq $0, _62
		jmp block77

	block77:
		movq free_ptr(%rip), %r11
		addq $16, free_ptr(%rip)
		movq $3, 0(%r11)
		movq %r11, alloc60
		movq alloc60, %r11
		movq $777, 8(%r11)
		movq $0, _61
		movq alloc60, a57
		movq free_ptr(%rip), tmp70
		movq tmp70, tmp71
		addq $24, tmp71
		movq fromspace_end(%rip), tmp72
		cmpq tmp72, tmp71
		jl block75
		jmp block76

	block76:
		movq %r15, %rdi
		movq $24, %rsi
		callq collect
		jmp block74

	block75:
		movq $0, _66
		jmp block74

	block74:
		movq free_ptr(%rip), %r11
		addq $24, free_ptr(%rip)
		movq $389, 0(%r11)
		movq %r11, alloc63
		movq alloc63, %r11
		movq a57, 8(%r11)
		movq $0, _65
		movq alloc63, %r11
		movq a57, 16(%r11)
		movq $0, _64
		movq alloc63, b58
		movq b58, %r11
		movq 8(%r11), tmp73
		movq tmp73, %r11
		movq $42, 8(%r11)
		movq $0, _59
		movq a57, %r11
		movq 8(%r11), %rax
		jmp conclusion

	start:
		movq free_ptr(%rip), tmp67
		movq tmp67, tmp68
		addq $16, tmp68
		movq fromspace_end(%rip), tmp69
		cmpq tmp69, tmp68
		jl block78
		jmp block79


## Register Allocation

Put spilled tuples on the root stack.

If a tuple is live during a call to collect, spill it.

Register Allocation:
    
	b58     %rcx
	_59     %rcx
	alloc60 %rcx
	_61     %rdx
	_62     %rcx
	alloc63 %rcx
	_64     %rdx
	_65     %rdx
	_66     %rcx
	tmp67   %rcx
	tmp68   %rcx
	tmp69   %rdx
	tmp70   %rcx
	tmp71   %rcx
	tmp72   %rdx
	a57     -8(%r15)
	tmp73   %rcx

Output:
    
	block79:
		movq %r15, %rdi
		movq $16, %rsi
		callq collect
		jmp block77

	start:
		movq free_ptr(%rip), %rcx
		movq %rcx, %rcx
		addq $16, %rcx
		movq fromspace_end(%rip), %rdx
		cmpq %rdx, %rcx
		jl block78
		jmp block79

	block78:
		movq $0, %rcx
		jmp block77

	block75:
		movq $0, %rcx
		jmp block74

	block74:
		movq free_ptr(%rip), %r11
		addq $24, free_ptr(%rip)
		movq $389, 0(%r11)
		movq %r11, %rcx
		movq %rcx, %r11
		movq -8(%r15), 8(%r11)
		movq $0, %rdx
		movq %rcx, %r11
		movq -8(%r15), 16(%r11)
		movq $0, %rdx
		movq %rcx, %rcx
		movq %rcx, %r11
		movq 8(%r11), %rcx
		movq %rcx, %r11
		movq $42, 8(%r11)
		movq $0, %rcx
		movq -8(%r15), %r11
		movq 8(%r11), %rax
		jmp conclusion

	block77:
		movq free_ptr(%rip), %r11
		addq $16, free_ptr(%rip)
		movq $3, 0(%r11)
		movq %r11, %rcx
		movq %rcx, %r11
		movq $777, 8(%r11)
		movq $0, %rdx
		movq %rcx, -8(%r15)
		movq free_ptr(%rip), %rcx
		movq %rcx, %rcx
		addq $24, %rcx
		movq fromspace_end(%rip), %rdx
		cmpq %rdx, %rcx
		jl block75
		jmp block76

	block76:
		movq %r15, %rdi
		movq $24, %rsi
		callq collect
		jmp block74
    
## Prelude and Conclusion

	main:
		pushq %rbp
		movq %rsp, %rbp
		subq $0, %rsp
		movq $65536, %rdi
		movq $65536, %rsi
		callq initialize
		movq rootstack_begin(%rip), %r15
		movq $0, 0(%r15)
		addq $8, %r15
		jmp start

    ...

	conclusion:
		subq $8, %r15
		addq $0, %rsp
		popq %rbp
		retq

# Functions

