---
layout: default
permalink: /asm-plugin/
title: Asm Plugin
---


The asm plugin is straightforward and allows an architecture's code to be assembled and dissembled. Most of the plugin code is placed into libr/asm/arch/, while the code placed into libr/asm/p/ is used for building the information structure and calling the main code. Below is an explanation of how I wrote the plugin. 

The first task is to create a struct representing the micro-controller's opcodes and how to process them. I placed this struct and array into MC81F4204_ops.h. 

{% highlight c %}
typedef struct {
    ut8 op;
    ut8 instr;
    char* string;
    ut8 len;
    ut16 flag;
    ut8 addr_mask
} _MC81F4204_op_t;
{% endhighlight %}

The _MC81F4204_op_t struct contains information such as the opcode, type of instruction, dissembled string, length of the opcode, flag representing special processing, and bit mask to get the opcode if applicable. Below is an example of the array representing the opcodes. 

{% highlight c %}
static _MC81F4204_op_t _MC81F4204_ops[] = {
    {0x04, OP_ADC, "adc A, 0x%02x", 2, NO_FLAGS, NO_MASK},
    {0x05, OP_ADC, "adc A, [rpr + 0x%02x]", 2, NO_FLAGS, NO_MASK}, 
    {0x06, OP_ADC, "adc A, [rpr + 0x%02x + X]", 2, NO_FLAGS, NO_MASK},
    {0x07, OP_ADC, "adc A, [0x%04x]", 3, NO_FLAGS, NO_MASK},
    {0x15, OP_ADC, "adc A, [0x%04x] + Y", 3, NO_FLAGS, NO_MASK},
    {0x16, OP_ADC, "adc A, [ [rpr + 0x%02x + X] ]", 2, NO_FLAGS, NO_MASK},
...
{% endhighlight %}

The actual dissemble code is in MC81F4204_disas.c, which will loop through the _MC81F4204_ops array looking for the current byte.

{% highlight c %}
int _MC81F4204_disas(ut64 pc, RAsmOp *op, const ut8 *buf, ut64 len) {
    int i = 0; // index of op in op array
    while (_MC81F4204_ops[i].string && _MC81F4204_ops[i].op != (buf[0] & _MC81F4204_ops[i].addr_mask)) {
        i++;
    } // search through array for current opcode

    if (_MC81F4204_ops[i].string) { // in op array
        const char* name = _MC81F4204_ops[i].string;
        ut8 oplen = _MC81F4204_ops[i].len;
        ut16 flag = _MC81F4204_ops[i].flag;
        char* disasm = 0;
...
{% endhighlight %}

If a match is found then the function will use information in the _MC81F4204_op_t struct to dissemble the instruction. Below is an example of the logic to dissemble an instruction that is two bytes long. 

{% highlight c %}
        switch (oplen) {
...
        case 2:
            if (len > 1) {
                if (flag == NO_FLAGS) {
                    disasm = r_str_newf(name, buf[1]);
                } else if (flag == REL_JMP) {
                    disasm = r_str_newf(name, rel_jmp_addr(pc + 2, buf[1]));
                } else if (flag == BYTE_BIT_POS) {
                    disasm = r_str_newf(name, buf[1] >> 5);
                } else if (flag == BIT_IN_OP) {
                    disasm = r_str_newf(name, buf[1], buf[0] >> 5);
                } else if (flag == BRANCH_BIT_IN_OP) {
                    disasm = r_str_newf(name, buf[0] >> 5, rel_jmp_addr(pc + 2, buf[1]));
                }
            } else {
                r_strbuf_set(&op->buf_asm, "truncated");
                return -1;
            }
            break;
...
{% endhighlight %}

The function r_str_newf provides a format string capability to radare2. The processing of each opcode can be as simple as taking the 2nd byte and inserting into the instruction's string. A more complicated example, is using part of the 1st byte to determine a bit from memory to check and branch to a relative location based on the result. 


[Example of opcode being decoded and dissembled]

{% highlight c %}
...
        }
        //TODO Change control register to names
        r_strbuf_set(&op->buf_asm, disasm);
        free(disasm);

        return oplen;
    }

    // invalid opcode
    return 0;
}
{% endhighlight %}

The last part of the _MC81F4204_disas function adds the dissembled opcode to radare2's opcode struct and returns the number of bytes processed. Radare2 calls this function over and over until there are no more bytes to dissemble. 


After the asm plugin is complete, the new architecture can be dissembled but not analyzed. 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/asm/dissembled.png)
