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

| Type         | Data Unit   | Size         | Demo                               | Formats        | TX ops        | RX ops                                   |
| ------------ | ----------- | ------------ | ---------------------------------- | -------------- | ------------- | ---------------------------------------- |
| large file   | file        | random       | sftp                               | ASCII/EBCDIC   | encryption    | decryption                               |
| text (NLP)   | sentence    | <1KB         | data-vectorization                 | ASCII          | none          | vectorization (lookup, matrix multiply.) |
| image        | image       | (500B, 20MB) | decision_tree based face detection | JPEG, PNG, GIF | compress      | decompress, binarization                 |
| video        | video chunk | (5MB, 50MB)  | MPEG-DASH                          | 264            | DASH proto    | DASH proto                               |
| JAVA objects | objects     | <128MB       | Cereal[2] / JAVA bulit-in          | byte sequence  | serialization | deserialization                          |
| undefined    |             |              |                                    |                |               |                                          |

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

#### DMA engine (a key part)

Instead of packet descriptors, the DMA engine needs to use the abstraction of ADU instead of packet for the communication between NIC and the memory.

### Software Design

The API here is mainly to replace the conventional Socket API. Before the NIC is able to process the ADU, certain instructions should be pushed to the NIC:

1. underlaying packet header info;
2. operation type and related metadata (encryption key, JAVA object metadata, etc.);
3. other info for ADU-packet conversion (e.g., packet size), scheduling priority, forwarding rules.

[API design considerations]

### Security Design

isolation between different ADUs/applications.



### Reference

[1] Architectural Considerations for a New Generation of Protocols.

[2] A Specialized Architecture for Object Serialization with Applications to Big Data Analytics.



 







