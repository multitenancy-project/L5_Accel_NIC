## Design Note 

According to [1], the most efficient way of processing data would be to push as many operations as possible to the data if its currently in the cache or registers. Which means the layered design doesn't make a lot of sense if there are lots of manipulations to the data from application layer all the way down to the network layer. 

> This requires re-design to both API (to replace socket) and the NIC.

the design of ADU should be an End-to-End design:

1. application pushes ADU (i.e. DMA address of the ADU) to the NIC;

   1. the length of ADU is a tradeoff. (video chunks, text message, image, file)

      > I need also consider that the memory on the NIC is limited.

   2. the sender side should provide location info of ADUs in case ADUs arrived out-of-order on the receiver side.

      > or at least the sender and receiver should have high-level agreement on where/when the ADU should be put.

2. NIC is responsible for the ADU to packet conversion and visa verse.

**The ADU-packet conversions must be driven by application knowledge.**

**ADU should take the place of packet for manipulation:**

* can be processed out of order;
* can be mapped onto memory (user space) separately;
* smallest unit for error recovery.



> 2 types of ADU delivery between NIC & memory:
>
> 1. push the data to memory blocks successively (file transfer);
> 2. copy the data into separate places (as results/parameters of program, RPC.) **hard to achieve.**

### Motivation with real-world demos

Two experiments showing the proportion of CPU cycles on the ADU generation (pre-processing). And using perf to obtain the `L3 cache missing rate` and `CPU cycles` spent on data manipulating operations.

1. Data Vectorization for NLP processing; 
   1. skip spaces and emojis from the text;
   2. convert the words to one-hot vectors using dictionary;
   3.  push the vectors into GRU (matrix multiply) and obtain the final vector.
2. word aligned copy from R2000 integer to ASN.1;
   1.  aligned copy to ASN.1 and word aligned.

### Expecting Performance Bottlenecks

	1. manipulations to the data (conversion, compress, encryption);
	2. random memory access (JAVA objection serialization, file transfer)

### ADU abstraction

As suggested in [1], the definition of ADU (contents, length) is based on application knowledge. There are several types of application that cover most of the services that servers provide in the cloud.

| Type         | Data Unit   | Size         | Demo                      | Formats        | TX ops                 | RX ops                                   | IP  cores                          |
| ------------ | ----------- | ------------ | ------------------------- | -------------- | ---------------------- | ---------------------------------------- | ---------------------------------- |
| large file   | file        | random       | sftp                      | ASCII/EBCDIC   | encryption/compression | decryption/decompression                 | https://github.com/linuxbest/lzs   |
| text (NLP)   | sentence    | <1KB         | data-vectorization        | ASCII          | none                   | vectorization (lookup, matrix multiply.) | https://github.com/mkearney/wactor |
| image        | image       | (500B, 20MB) | face-detection            | JPEG, PNG, GIF | compression            | decompression, binarization              | vivado  built-in                   |
| video        | video chunk | (5MB, 50MB)  | MPEG-DASH                 | 264            | H264 encoding          | H264 decoding                            | vivado  built-in                   |
| JAVA objects | objects     | <128MB       | Cereal[2] / JAVA bulit-in | byte sequence  | serialization          | deserialization                          |                                    |
| undefined    |             |              |                           |                |                        |                                          |                                    |

According to the table above, the size of ADU should range from several types to several hundreds of MB. 

**E2E Data Execution Path**: For each type of application:

1. on the sender side, the CPU should push the pointer which points to the physical address of the data (can be DMAed) to the NIC together with the metadata (how the ADU should be pre-processed);
2. The specific ADU-module pulls the ADU from DMA and execute the TX ops (encryption, compress, serialization, etc.);
3. (ADUs are converted into packets and transferred in the network)
4. On the receiver side, the RX ops (decryption, vectorization, decompression) that a specific ADU-module should execute are registered right after the application started up;
5. After the the packet is dispatched from RMT pipeline, the ADU-module obtains packets and re-construct the ADU. Then execute the operations registered before. 

### Hardware Design

![image-20210317230039197](fig/architecture_draft.png)

#### Pipeline Design (the figures above)

Because of the asymmetric feature of the data path, its easier to have separate TX/RX paths on the NIC.

##### RMT pipeline

The switch fabric on the NIC. Beside basic switch services, in this project, the RMT pipeline can be used to **reformatting packet headers** for each packet on the TX path. For example, the ADU sends out the payload wrapped in a **fake packet header** (using a same length of `EtherHdr`+`IPHdr`+`UDPHdr`, but just contains an internal ADU ID). Then the RMT pipeline is responsible for replacing the ADU index with the correct packet header that the user configured using p4.

On the RX path, the RMT pipeline will re-direct the packet to the determined ADU module according to the rules offloaded by the user by his p4 code.

##### Scheduler

Scheduler is connected with ADU modules and Dispatch module using a flexible crossbar (with a simplified switch table instead). On the TX path, Scheduler is responsible for scheduling ADUs to different ADU modules. On the other hand, Scheduler is implemented as a typical **N to 1** scheduler to push ADUs back to the host. 

Beside ADUs, the scheduler needs also deliver DMA requests (read & write) from ADU modules to the DMA engine. 

##### Dispatch

The dispatch module directly connects to the DMA engine which is used to write/read data to/out of the main memory. On the TX path, it receives DMA read request from ADU modules (time division multiplexed) and reads the requested data from the main memory (via DMA), then reads ADUs from DMA engine to the scheduler (with the metadata related). 

On the RX path, it passes DMA write request from ADU module to the DMA engine and write the ADU back to the main memory.

##### ADU Modules (multi-queue DMA)

**each ADU module contains an `address management unit` for data delivery between NIC and the main memory.**

ADU module is the key module which performs the data pre-processing to the raw data received from DMA or the packet from network interface. There are a couple of operations ADU modules need to perform on the RX/TX paths separately.

On the RX path:

1. ADU reformatting: reorder the packet, deprive the packet header and reformat the ADU in a form that can be processed directly by the ADU module;
2. ADU processing: computing tasks include compression/decompression, decryption/encryption, serialization/deserialization, encoding/decoding, etc.
3. DMA write issuing: depending on different types of data-preprocessing, the ADU module issues different types of DMA write (block or scatter-gather).

On the TX path:

1. ADU processing: computing tasks include compression/decompression, encryption/decryption, serialization/deserialization, encoding/decoding, etc.;
2. DMA read issuing: depending on different types of data-preprocessing and the remained address in the **address management unit**, the ADU module issues different types of DMA read to the main memory.



**Queue Management**: In order to support isolated multiple ADUs processing on the NIC, a multi-queue DMA engine is needed. The high-level DMA design can be shown in the figure below:

![DMA_design](fig\DMA.png) 

The key feature of the DMA is that: **each ADU module has its own descriptor queue that it can operate with.** the MUX module schedules DMA requests from each ADU module and issues DMA transactions to the memory. On the software side, **each application has separated buffer in the kernel (can be mapped to user-space by `mmap()` system call)**. In this way, the ADU of each application is completely isolated (if everyone behave well).

**Descriptor Design**: The descriptor needs to include info such as address, data size, ADU ID, etc. Thus, a simple format we can use for descriptor is:

![desc](fig/desc.png)

On the software, for each block of data that needs to be transferred to the NIC, the application process needs to fill the `simple_descrptor` node on the DMA-able space:

```C
struct simple_desc{
	_u8  ADU_ID;
	_u64 data_addr;
	_u32 data_len;
	_u16 resv;
};
```



### Software Design

#### ADU Manager

To work with NIC properly, the software side should also maintain the data path that ADUs from a specific application should go through. **ADU Manager** is responsible for data path configuration in this regard. It works as an independent process. The data path determinations of TX/RX path have different considerations:

For the TX path, which ADU module that a block of data feed into is directed by the scheduler. Thus, its clear that the software side should be able to config a **redirect table** (implemented on the NIC) before firing up the application (who has a corresponding ADU module on the NIC). The format of the table entry should be like: |  ADU_ID  |  Channel_ID   |. Considering the ADU is tightly coupled with a specific application on the software, its reasonable that we use **16b L4 port ID** as the **ADU_ID **. And the **8b Channel_ID** is the **sequence number of  a specific ADU module** on the NIC. Thus, the scheduler is able to redirect the ADU correctly to the ADU module.

![image-20210325201744485](fig/redirect_table.png)

For the RX path, the data path is decided by the RMT pipeline. Thus, its natural to use p4 to control the data path of the ADU coming from the network. Same as the TX path, there is also a **redirect table** (implemented in RMT pipeline), which entry has two similar fields: | ADU_ID  |  Channel_ID |. The **16b ADU_ID** is actually the **port ID on L4**.

#### Application

The application on the software runs exact the same as applications running on conventional servers, except that it doesn't use socket for data send/receive.

#### SW-HW Interface

In order to perform the ADU processing correctly on the NIC, certain API/data structures shall be used to config the switch pipeline and ADU modules as needed. For  

The API here is mainly to replace the conventional Socket API. Before the NIC is able to process the ADU, certain instructions should be pushed to the NIC:

1. underlaying packet header info, so that the NIC can wrap up ADUs in packets (take the place of network stack in the kernel);

2. the info needed by the ADU module to process ADUs on the NIC;

   > keys for encryption/decryption, img size for img compression/decompression, dictionary for word vectorization, etc. 

3. other info for ADU-packet conversion (e.g., packet size), scheduling priority, forwarding rules.



### Security Design

isolation between different ADUs/applications.

### Design Considerations

##### 1. How the ADU can be written to DMA-able buffer space? 

OK, things are a little bit tricky. Even with a user-space DMA mechanism, the buffer space still needs to be issued in the kernel and mapped onto user-space. This means that ADUs still need one-copy before it can be DMAed to the NIC. On the RX path, the ADU still needs to be copied out of the buffer space to the user process.

##### 2. How the NIC works with partial reconfiguration?

since the NIC hasn't to be fully implemented using FPGA, partial reconfiguration is only optional here. However, the skeleton of the NIC should be FPGA based, which makes partial reconfiguration possible for a FPGA-based ADU module.

### Reference

[1] Architectural Considerations for a New Generation of Protocols.

[2] A Specialized Architecture for Object Serialization with Applications to Big Data Analytics.



 







