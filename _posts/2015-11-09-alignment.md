---

layout: post

title: An Unexpected Cause of an Alignment Fault

---



I’d like to start off this blog by describing a bug that we ran into a few weeks ago. The details are specific to our processor type, but I think the lessons are somewhat more general. The bug also showed up in an interesting and unexpected way.

### The Setup

If you write software, at some point you’ve probably had to serialize and deserialize objects. Libraries like [protobuf](https://developers.google.com/protocol-buffers/?hl=en) are great for handling this. But these kinds of libraries are not always available, and sometimes you have to roll your own. In our case, we hadn’t even gone that far, and just had some deserialization code that looked like this:

```c
void deserializeFoo(uint8_t *packet, Foo *foo) {
  uint32_t arg1 = *((uint32_t*) (packet);
  uint32_t arg2 = *((uint32_t*) (packet + 4);
  float32_t arg3 = *((float32_t*) (packet + 8);
  //...
  foo->field1 = arg1;
  //...
}
```

OK so this isn’t great. But If you’ve spent time writing native code, you’ve probably seen this pattern, and perhaps used it yourself. I consider this an anti-pattern<sup>1</sup> for a couple of reasons: it depends on the endianness of the host system (which makes it hard to port), and it can lead to alignment faults on certain architectures. I won’t dwell on this too much. If you are interested, [here](http://commandcenter.blogspot.com/2012/04/byte-order-fallacy.html) is a good writeup. But I do want to write up an error that we saw, because it showed up in a pretty unexpected way.

First, some background (**update: this company is sadly now defunct**). We’re building a set of wireless earbuds (shameless plug: [https://www.hereplus.me/](https://www.hereplus.me/)) that let you control how you hear the world. Users can control the earbuds through an iOS app, which sends command messages to the buds. The details aren’t important to this bug, but you can think of the interface as a set of endpoints, each with an associated handler that processes the incoming byte array. These handlers call into the deserialization code, almost all of which used the above pattern.

For a while, everything worked fine. Our processor is a little-endian ARM Cortex-M4, so no issues with endianness. Alignment faults might present a problem, but the incoming packets are all 4-byte aligned, so no problems there.  Then, one day, when sending commands to one of the endpoints, the processor would go into a hard fault. Luckily, this issue was very easy to reproduce, and we were quickly able to isolate the breaking change. Here it is (with some minor edits):

```c
void deserializeFoo(uint8_t *packet, Foo *foo) {
  // Parse the id
  uint32_t id = *((uint32_t*) packet);
  // Parse the value
  float32_t value = *((float32_t*) (packet + 4));
  // Hydrate the object
  foo->id = id;
- foo->value = value;
+ // Incoming values are all scaled by 0.5
+ foo->value = 2.f * value;
}
```

Umm… That sure doesn’t look like it would cause an alignment fault. But sure enough, the fault shows up with that change, and goes away without it. And the fault register clearly shows an alignment fault.

### What Happened

The error was really caused by a combination of two changes. In the first, one of our developers combined two of the endpoints into a single one.<sup>2</sup>

```c
void handleFooBarPacket(uint8_t *packet) {
  switch (packet[0]) {
    case FOO: {
      Foo foo;
      deserializeFoo(packet + 1, &foo);
      //...
    } break;
    case BAR: {
      Bar bar;
      deserializeBar(packet + 1, &bar);
      //...
    } break;
  //...
  }
}
```

In order to combine the two endpoints, the developer changed the packet protocol to contain a type value as the first byte, and shifted the rest of the packet down a byte. What’s surprising to me is that this change didn’t break anything. I vaguely remembered reading that the Cortex-M4 supports unaligned accesses for non-floating point values. But **all** of the parameter parsing (including floating point accesses) results in unaligned accesses! And why did it work until one of the parameters was multiplied by 2?

At this point, we were pretty much stuck with looking at the compiler output. Here’s the old, working output for deserializeFoo (general purpose registers are named `r*`, while floating point registers are named `s*`):

```assembly
deserializeFoo:
...
; r3 holds a pointer to a Foo
ldr r1, [r0]
ldr r2, [r0, #4]
str r1, [r3]
str r2, [r3, #4]
...
bx lr
```

And the broken output:

```assembly
deserializeFoo:
...
; r3 holds a pointer to a Foo
ldr r1, [r0]
vldr s0, [r0, #4]
vmul s0, s0, s1 ; s1 is a register with the constant 2 in in
str r1, [r3]
vstr s0, [r3, #4]
...
bx lr
```



The change from [LDR](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0553a/BABJGHFJ.html) (used to load a value into a general purpose register) to [VLDR](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0553a/CHDICEDI.html) (used to load a value into a floating point register) is pretty suggestive. We stepped through the code in a debugger, and the fault definitely occurred on the `VLDR`. The issue is actually pretty subtle. The Cortex-M4 chips (can) support unaligned access for the `LDR` instruction, but not for the `VLDR` instruction.<sup>3</sup> When no intermediate computations were applied to the variable, the compiler figured that it could just generate code to read the (floating point) value into a general purpose register, and then write it out to memory. It only generated a load into a floating point register once it needed to double the value before writing it out. In retrospect, this shouldn’t have surprised me. There’s nothing there that would require a load into a floating point register. I guess I was wired to think that any operation involving a float occurred in a floating point register.

### Lessons

I drew a few lessons here. First and most prosaically, grab a library for serialization. If you can’t due to code size constraints, roll your own quick one whose correctness you can easily reason about (for example, that reads one byte at a time and assembles larger primitives using bitshifts). Unless you have a good reason to, don’t rely on fragile constructs like:

```c
*((T*) byte_array)
```

Another is that the ultimate and proximate causes of errors can be very far apart. In the end, the culprit was the alignment fault that we expected, but the compiler had masked it for a while by avoiding unnecessary floating point loads. This bug could have remained latent in our codebase forever. And last, types are a language-level construct. Just because a value is declared float doesn’t mean that it will be stored in a floating point register.

#### Footnotes

1. Thinking a bit more, I can imagine performance sensitive applications (for example, fast compressors / decompressors) where we would want to avoid byte-by-byte access, but let’s ignore those for now.
2. Endpoints (characteristics, in Bluetooth-speak) are pretty expensive to add, so consolidating them freed up some Bluetooth resources that we needed.
3. If you’re interested, this is covered in section A3.2.1 of the ARMv7 architecture reference manual.
