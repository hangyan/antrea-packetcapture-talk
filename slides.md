---
# You can also start simply with 'default'
# theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://cover.sli.dev
background: https://source.unsplash.com/collection/94734566/1920x1080
# some information about your slides (markdown enabled)
title: Antrea Packetcapture Introduction
info: |
  ## Slidev Starter Template
  API, DataPath, and Follow-up.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true
colorSchema: light
---

# Antrea PacketCapture Introduction

API, Datapath, and Follow-up

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/antrea-io/antrea" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->


---
transition: fade-out
layout: two-cols
layoutClass: gap-16
zoom: 0.8
---


# Current API

```yaml  {all|2|6-7|9-11|13-15|23|all} twoslash
apiVersion: crd.antrea.io/v1alpha1
kind: PacketCapture
metadata:
  name: pc-test
spec:
  fileServer:
    url: sftp://127.0.0.1:22/upload # Define your own sftp url here.
  timeout: 60
  captureConfig:
    firstN:
      number: 5
  source:
    pod:
      namespace: default
      name: frontend
  destination:
  # Available options for source/destination could be `pod` (a Pod), 
  # `ip` (IP address). These 2 options are mutually exclusive.
    pod:
      namespace: default
      name: backend
  packet:
    ipFamily: IPv4
	# support arbitrary number values and string values 
	# in [TCP,UDP,ICMP] (case insensitive)
    protocol: TCP 
    transportHeader:
      tcp:
        dstPort: 8080 
```
::right::

# Original Version

```yaml
apiVersion: crd.antrea.io/v1alpha1
kind: PacketSampling
metadata:
  name: ps-test
spec:
  timeout: 60             
  type: FirstNSampling    
  parameters: 
    number: 15            
  source:                 
    namespace: default
    pod: tcp-sts-0
  destination: 
    namespace: default
    pod: tcp-sts-2   
  packet:
    ipHeader: 
      protocol: 6 
    transportHeader:
      tcp:
        srcPort: 10000 
        dstPort: 80 
  fileServer:             # storage part
    url: sftp://youtestdomain.com:22/root/test
  authentication:
    authType: “BasicAuthenticaion“
    authSecret:
      name: support-bundle-secret
      namespace: default

```

---
transition: slide-up
level: 2
zoom: 0.85
---

# Datapath Options

| SOLUTION               | DESCRIPTION                                                  | PROS                                          | CONS                                                 |
|:----------------------:|:------------------------------------------------------------:|:---------------------------------------------:|:----------------------------------------------------:|
| eBPF                   | attach ebpf program                                          | popular / high performance / active community | whole new teck stack/require certain kernel versions |
| ovs                    | use existing antrea pipeline                                 | built-in / async (packet-in)                  | a bit of complicated for this case                   |
| libbpf+gopacket        | use libpf library and go interface                           | well tested                                   | CGO required, a lot of ci changes                    |
| third-party go bpf library | pure go implemention of bpf                                  | easy to use                                   | bugs, not much choice                                |
| our own bpf library ✅ | pure go implemention of bpf, but only a subset of the filter | not hard to implement and maintain            | extra concurrent layer around k8s controller         |



---
transition: slide-up
level: 2
zoom: 0.8
layout:  two-cols
---



# Example

Say i want to capture tcp traffic between two pods:

<div v-click>

it's equivalent <code>tcpdump</code> expression would be

```bash
tcpdump  -i <dev> "ip proto 6 and src host 10.244.2.3 \
and dst host 10.244.1.3 and dst port 80"
```

</div>

<br>

<v-click>

An extra <code> -d </code> option would dump the following bpf instructions:
```asm
(000) ldh      [12]
(001) jeq      #0x800           jt 2	jf 14
(002) ldb      [23]
(003) jeq      #0x6             jt 4	jf 14
(004) ld       [26]
(005) jeq      #0xaf40203       jt 6	jf 14
(006) ld       [30]
(007) jeq      #0xaf40103       jt 8	jf 14
(008) ldh      [20]
(009) jset     #0x1fff          jt 14	jf 10
(010) ldxb     4*([14]&0xf)
(011) ldh      [x + 16]
(012) jeq      #0x50            jt 13	jf 14
(013) ret      #262144
(014) ret      #0
```
</v-click>


::right::

<v-click>

# Golang Implemention

translate the bpf instructions generated by `tcpdump` to go code:

```go
import (
	"golang.org/x/net/bpf"
)

func generateBPFInstructions() {
	 bpf.LoadAbsolute{Off: 12, Size: lengthHalf}
	 bpf.LoadAbsolute{Off: 23, Size: lengthByte}
	 bpf.JumpIf{
		 Cond: bpf.JumpEqual, 
		 Val: protocol, 
		 SkipTrue: skipTrue, 
		 SkipFalse: skipFalse
	 }
	 
	 bpf.LoadAbsolute{Off: 26, Size: lengthWord}
	 addrVal := binary.BigEndian.Uint32(dstIP[len(dstIP)-4:])
	 bpf.JumpIf{
		 Cond: bpf.JumpEqual, 
		 Val: addrVal, 
		 SkipTrue: 0, 
		 SkipFalse: size - uint8(len(inst)) - 2
	 }
	 ...
}
```
</v-click>

---
layout: image
image: /images/packet.jpg
zoom: 1
backgroundSize: contain
---

---

# What's Next

- **Bi-Direction Capture** : source -> destination and destination -> source
- **ICMP echo/reply filter**
- **TCP flags** : for example `'tcp[13] & 16 != 0'` (all ack packets)
- **IPv6**
- ...
- **1 of N Sampling**: [https://www.kernel.org/doc/html/v6.1/networking/filter.html](https://www.kernel.org/doc/html/v6.1/networking/filter.html)

```asm
ldh [12]
jne #0x800, drop
ldb [23]
jneq #1, drop
ld rand # get a random uint32 number
mod #4  # 1 out of 4
jneq #1, drop
ret #-1
drop: ret #0
```


generate the bpf instructions using `tcpdump -d` and then translate them into golang code.


---
layout: center
class: 'text-center pb-5 :'
---

# Thank You !

Welcome to try this feature out in the latest antrea release.

[https://github.com/antrea-io/antrea/releases/tag/v2.2.0](https://github.com/antrea-io/antrea/releases/tag/v2.2.0)
