---
layout: default
permalink: /bin-plugin/
title: Bin Plugin
---

The bin plugin allows radare2 to recognize new file formats that an architecture uses. When a file is opened, radare2 uses the bin plugins to determine who to read that file. Radare2 determines which bin plugin to use based on the file's magic number. A bin plugin must have a check_bytes function to check the loaded files' magic number. 

{% highlight c %}
static bool check_bytes(const ut8 *buf, ut64 length) {
	if (!buf || length < 3) {
		return false;
	}
	return (!memcmp (buf, MC81F4204_MAGIC, 3));
}
{% endhighlight %}

The check_bytes function ensures the file is greater than 3 bytes, and then checks if the first 3 bytes are "\x60\x1e\x00". 

The bin plugin can also define virtual address scheme, sections, symbols and entry points for an architecture based on the file format. 

Virtual addressing requires two things, the has_va flag to be set in the RBinInfo struct and a section that maps the physical address to virtual address. 

{% highlight c %}
static RBinInfo *info(RBinFile *bf) {
	RBinInfo *ret = NULL;
	..
	ret->arch = strdup ("MC81F4204");
	ret->bits = 8;
	ret->has_va = 1;
	return ret;
}
{% endhighlight %}

The sections function uses the below _MC81F4204_sections_table to build the sections for the MC81F4204. The MC81F4204 has the firmware mapped to the memory address 0xF000, and this is what these sections do. The section "ROM" is located in the file at offset 0x0, but the virtual address is at 0xF000. 
 
{% highlight c %}
typedef struct {
	char* name;
	ut16 paddr;
	ut16 size;
	ut16 vaddr;
	ut16 vsize;
} _MC81F4204_section;


static _MC81F4204_section _MC81F4204_sections_table[] = {
	{"ROM", 0x0, 0x0EFF, 0xF000, 0x0EFF},
	{"PCALL_TABLE", 0x0F00, 0xC0, 0xFF00, 0xC0},
	{"TCALL_TABLE", 0x0FC0, 0x20, 0xFFC0, 0x20},
	{"INTERRUPTS", 0x0FE0, 0x20, 0xFFE0, 0x20}
};
{% endhighlight %}



{% highlight c %}
static RList* symbols(RBinFile *bf) { // make array and loop
	RList *ret = NULL;
	if (!(ret = r_list_new ())) {
		return NULL;
	}

	for (ut8 i = 0; i <= 18; i++) {
	    RBinSymbol *ptr = NULL;
		if (!(ptr = R_NEW0 (RBinSymbol))) {
			return ret;
		}

		ptr->name = r_str_new(_MC81F4204_symbols_table[i].name);
		ptr->paddr = _MC81F4204_symbols_table[i].addr - 0xF000;
        ptr->vaddr = _MC81F4204_symbols_table[i].addr;
		ptr->size = _MC81F4204_symbols_table[i].size;

		r_list_append (ret, ptr);
	}

	return ret;
}
{% endhighlight %}

The symbols function will build a list of RBinSymbol structs, which contain the name, size, physical and virtual address of each symbol. 


{% highlight c %}
static RList* sections(RBinFile *bf) {
	RList *ret = NULL;
	
	if (!(ret = r_list_new ())) {
		return NULL;
	}


	for (ut8 i = 0; i <= 3; i++) {
	    RBinSection *ptr = NULL;
		if (!(ptr = R_NEW0 (RBinSection))) {
			return ret;
		}

		ptr->name = r_str_new(_MC81F4204_sections_table[i].name);
		ptr->paddr = _MC81F4204_sections_table[i].paddr;
		ptr->size = _MC81F4204_sections_table[i].size;
		ptr->vaddr = _MC81F4204_sections_table[i].vaddr;
		ptr->vsize = _MC81F4204_sections_table[i].vsize;
		ptr->perm = R_PERM_RX;
		ptr->add = true;
		r_list_append (ret, ptr);
	}

	return ret;
}
{% endhighlight %}

The section function will do the same, but building a RBinSection struct, containing: name, physical address, physical size, virtual address, virtual size, read permissions, and if it should be added. 


{% highlight c %}
static RList* entries(RBinFile *bf) { 
	RList *ret;
	RBinAddr *ptr = NULL;
	ut16 start_addr;
	memset (&start_addr, 0, 2);

	if (!(ret = r_list_new ())) {
		return NULL;
	}
	if (!(ptr = R_NEW0 (RBinAddr))) {
		return ret;
	}

	r_buf_read_at (bf->buf, RESET_VECTOR_ADDRESS_PHYSICAL, (ut8*)&start_addr,2);
	ptr->vaddr = start_addr;
	ptr->paddr = start_addr - 0xF000;
	r_list_append (ret, ptr);

	return ret;
}
{% endhighlight %}

The entries function will define the entry point for code. The entry point for the MC81F4204 is stored at the address 0x0FFE and is two bytes long. This function uses r_buf_read to that location in the file and store it into start_addr. 


To determine the magic number of the MC81F4204's firmware, I dissembled multiple examples of compiled code. I found that at the beginning of the firmware is used to copy data values from ROM to RAM. Even if there is no memory there to move, this code still exists. 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/bin/magic_number.png)

Now whar radare2 loads the firmware for the MC81F4204, it will automatically create sections, symbols and entry point. 

![_config.yml]({{ site.baseurl }}/images/mc81f4204/bin/bin_plugin.png)

