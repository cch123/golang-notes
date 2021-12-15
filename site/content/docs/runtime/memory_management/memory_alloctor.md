---
title: 内存分配
weight: 10
---

# 内存分配

## bump/sequential allocator

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D175%253A118%26node-id%3D175%253A119%26viewport%3D-7626%252C499%252C0.4998303949832916%26scaling%3Dcontain%26starting-point-node-id%3D175%253A119" allowfullscreen></iframe>

{{</rawhtml>}}

## freelist allocator

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D233%253A21%26node-id%3D233%253A22%26viewport%3D-650%252C182%252C0.059255365282297134%26scaling%3Dcontain" allowfullscreen></iframe>
{{</rawhtml>}}

## dangling pointer

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D175%253A16%26node-id%3D175%253A17%26viewport%3D-603%252C359%252C0.22722375392913818%26scaling%3Dmin-zoom%26starting-point-node-id%3D175%253A17" allowfullscreen></iframe>
{{</rawhtml>}}


## tiny alloc

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D165%253A81%26node-id%3D165%253A197%26viewport%3D-20163%252C1534%252C2.7550384998321533%26scaling%3Dscale-down" allowfullscreen></iframe>
{{</rawhtml>}}