---
layout: default
permalink: /analyze-plugin/
title: Analyze Plugin
---

The analyze plugin are stored in libr/anal/p and provides two features; the ability to analyze the new architecture and to emulate each instruction with ESIL. 

The function _MC81F4204_op will build a struct, RAnalOp, that radare2 will use to further analyze the code in a generic way. RAnalOp contains information such as: opcode size, address, type, cycles, and esil. Radare2 uses the type to determine how that instruction should be analyzed. For an example, the type R_ANAL_OP_TYPE_CJMP tells radare2 that this opcode creates a branch in the program flow. The type R_ANAL_OP_TYPE_CALL is indicates that the opcode will pass execution to a function. 

{% highlight c %}
static int _MC81F4204_op(RAnal *anal, RAnalOp *op, ut64 addr, const ut8 *data, int len, RAnalOpMask mask) {
	memset(op, '\0', sizeof(RAnalOp));
	op->size = 1;
	op->addr = addr;
	op->type = R_ANAL_OP_TYPE_UNK;
	op->id = data[0];
	int i = 0;
	r_strbuf_init(&op->esil);

	while (_MC81F4204_ops[i].op && _MC81F4204_ops[i].op != data[0]) {
		i++;
	} // search through array for current opcode esil data
	
	op->cycles = _MC81F4204_ops[i].cycles;
	op->size = _MC81F4204_ops[i].len;
...
{% endhighlight %}

This function uses a switch to set the opcode type, instead of a look up table. This allows more flexibility for any extra processing that needs to happen, such as for jump and call instructions. 

{% highlight c %}
switch(data[0]) {
...
	// ADC
	case 0x04:
	case 0x05:
	case 0x06:
	case 0x07:
	case 0x15:
	case 0x16:
	case 0x17:
	case 0x14:
		op->type = R_ANAL_OP_TYPE_ADD;
		break;
...
	// BEQ
	case 0xF0:
		op->type = R_ANAL_OP_TYPE_CJMP;
		op->jump = rel_jmp_addr(addr + op->size, data[op->size - 1]);
		op->fail = addr + op->size;
		break;
...
	// CALL !abs
	case 0x3B:
		op->type = R_ANAL_OP_TYPE_CALL;
		op->stackop = R_ANAL_STACK_INC;
		op->stackptr = 2;
		op->jump = _MC81F4204_determine_jmp_addr(data[1], data[2]);

		// TODO call code
		break;
...
{% endhighlight %}

The branch instruction requires the fail and jump values to be set. The fail value tells radare2 where the fail branch leads to, and jump is the address if the compare is true. The call instruction needs the stackop, stackptr, and jump values to be set. Stackop tells radare2 that this opcode effects the stack.

Enabling Esil

ESIL requires initialization and finalization functions, a register profile and each opcode to have ESIL instructions. ESIL is a stack based language, where operations are carried out by pushing values or operators on to a stack. For an example, to compute "4 + 7" the values 4 and 7 would be pushed to the stack. Next, the operator "+" would be pushed, this would cause the 4 and 7 to be popped from the stack, added and the result pushed back onto the stack. 

The initialization function sets up the starting values for all the registers. 

{% highlight c %}
static int esil_MC81F4204_init (RAnalEsil *esil) {
	if (esil->anal && esil->anal->reg) {		//initial values
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "pc", -1), 0x0000);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "sp", -1), 0xaf);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "a", -1), 0x00);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "x", -1), 0x00);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "y", -1), 0x00);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "psw", -1), 0x00);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "flags", -1), 0x00);
		r_reg_set_value (esil->anal->reg, r_reg_get (esil->anal->reg, "rpr", -1), 0x00);
	}
	return true;
}
{% endhighlight %}


A register profile is created with the function set_reg_profile. Each entry contains the register type, name, size, and position. The size and position fields allow registers to overlap and be referred to using multiple names. Any whole number refers to the byte position while the decimal value refers to the bit position. The flag register starts at the 4th byte and is 8 bits long. The C register represents the C flag and starts on the 32nd bit (4 * 8) which is the first bit of the flag register. 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/esil/reg_layout.png)

{% highlight c %}
static int set_reg_profile(RAnal *anal) {
	char *p =
		"=PC	pc\n"
		"=SP	sp\n"
		"gpr	x	.8	0	0\n"
		"gpr	ya	.16	1	0\n"
		"gpr	y	.8	1	0\n"
		"gpr	a	.8	2	0\n"
		"gpr	psw	.8	3	0\n"

		"gpr	flags	.8	4	0\n"
		"gpr	C	.1	.32	0\n"
		"gpr	Z	.1	.33	0\n"
		"gpr	I	.1	.34	0\n"
		"gpr	H	.1	.35	0\n"
		"gpr	B	.1	.36	0\n"
		"gpr	G	.1	.37	0\n"
		"gpr	V	.1	.38	0\n"
		"gpr	N	.1	.39	0\n"
		"gpr	sp	.8	4	0\n"
		"gpr	pc	.16	5	0\n"
		"gpr	pcl	.8	5	0\n"
		"gpr	pch	.8	6	0\n"
		"gpr	rpr	.8	7	0\n"
		"gpr	t	.8	8	0\n"; // temp register to make things easier
	return r_reg_set_profile_string (anal->reg, p);
}
{% endhighlight %}

The ESIL instructions are stored in a string with a comma separating each value/operator pushed onto the stack. The ESIL instruction needs to completely emulate the opcode, including setting the appropriate flags. Below is an example of the ESIL for the DIV instruction. 

{% highlight c %}
...
	// DIV
	case 0x9B:
		op->type = R_ANAL_OP_TYPE_DIV; // TODO, need to check
		r_strbuf_set (&op->esil, "ya, x, %, t, =, ya, x, /, a, =");
		_MC81F4204_anal_update_flags(op, _MC81F4204_FLAGS_NVHZ);
		r_strbuf_append (&op->esil,  "t, y, =");
		break;
...
{% endhighlight %}

![_config.yml]({{ site.baseurl }}/images/mc81f4204/esil/esil_example.png)

Radare2 will now show jump logic

![_config.yml]({{ site.baseurl }}/images/mc81f4204/analyze/jmps_working.png)

and is able to analyze the code using aa (analyze all) or aac (analyze all calls). 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/analyze/analyze_funcs.png)

Graph view is also more useful. 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/analyze/graph_view.png)
