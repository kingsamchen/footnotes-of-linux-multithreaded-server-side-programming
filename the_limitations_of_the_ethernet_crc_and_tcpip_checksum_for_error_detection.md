# The Limitations of the Ethernet CRC and TCP/IP checksums for error detection

Everyone knows that if you use TCP to transfer data across the Internet any corrupted TCP segment is detected and retransmitted. Well like so many other things, what everyone knows is not always correct.

The Ethernet frame check sequence check (FCS) and the IP and TCP checksums will protect your data from most but not all types of data corruption. This article will outline the types of errors that will not be detected.

The bottom line is that for any truly critical data you should either encapsulate the data in some form that will detect any corruption when you decapsulate it or at the very least transfer a hash (MD5, SHA-1, etc) of the data to confirm that the data has not been corrupted - or both.

## The Limitations of Ethernet Frame Check Sequence

The Ethernet Frame Check Sequence (FCS) is a 32 bit CRC. The mathematical analysis of CRC error detection is a complex subject and I will not go into it here. Suffice to say that the Ethernet FCS will detect

- Any 1 bit error
- Any two adjacent 1 bit errors
- Any odd number of 1 bit errors
- Any burst of errors with a length of 32 or less

Everyone agrees on the above but things become more nebulous when talking about bursts longer than 32 bits. Everyone agrees that some extremely small number of errors will still go undetected but actual numbers are hard to come by and no one seems to agree with anyone else.

Part of the problem might be the term "error burst". An error burst is defined by 2 values. First is the number of bits between the first and last error bits, for example a Y bit error burst will have bit N and bit N+Y-1 in error. Second is the value of the guard band, this is the number of contiguous bits within those Y bits that can be correct. None of the references that I found mentioned the value of the guard band.

Despite the vagueness of the error burst definition it would appear that the Ethernet CRC will detect the vast majority of errors. Unfortunately, "vast majority" is not "all". In addition, that majority is not as vast as the mathematics would lead you to believe. The problem is that the Ethernet FCS is recalculated by every Ethernet device between the source and destination. The calculation is done either by the Ethernet driver or on the chip itself. Any errors in the higher layer software of these devices or transient failures of the hardware (memory or bus) will result in the destination seeing an Ethernet frame with a valid FCS but containing corrupt data. To protect against these errors TCP is dependent on the IP and TCP checksums that are part of the protocol headers.

## The Limitations of the TCP and IP checksums

The IP checksum is a 16 bit 1's complement sum of all the 16 bit words in the IP header. Note that this does not cover the TCP header or any of the TCP data. The TCP checksum is a 16 bit 1's complement sum of all the 16 bit words in the TCP header (the checksum field is valued as 0x00 0x00) plus the IP source and destination address values, the protocol value (6), the length of the TCP segment (header + data), and all the TCP data bytes. If the number of header+data bytes is not an integer multiple of 16, pad bytes of 0x00 are added at the end until it is a multiple of 16. These pad bytes are not transmitted.

The checksum calculation will NOT detect

- Reordering of 2 byte words, i.e. 01 02 03 04 changes to 03 04 01 02
- Inserting zero-valued bytes i.e. 01 02 03 04 changes to 01 02 00 00 03 04
- Deleting zero-valued bytes i.e. 01 02 00 00 03 04 changes to 01 02 03 04
- Replacing a string of sixteen 0's with 1's or 1' with 0's
- Multiple errors which sum to zero, i.e. 01 02 03 04 changes to 01 03 03 03

In "Performance of Checksums and CRCs over Real Data"1 Stone and Partridge estimated that between 1 in 16 million and 1 in 10 billion TCP segments will have corrupt data and a correct TCP checksum. This estimate is based on their analysis of TCP segments with invalid checksums taken from several very different types of networks. The wide range of the estimate reflects the wide range of traffic patterns and hardware in those networks. One in 10 billion sounds like a lot until you realize that 10 billion maximum length Ethernet frames (1526 bytes including Ethernet Preamble) can be sent in a little over 33.91 hours on a gigabit network (10 * 10^9 * 1526 * 8 / 10^9 / 60 / 60 = 33.91 hours), or about 26 days over a T3.

## Am I being Paranoid?

So, if there is an undetectable corrupted segment on the network once every 34 hours or even once a month, why aren't the databases hopelessly corrupt? Why aren't applications randomly failing? Well, in many cases a corrupt segment is no big deal. For example, a bad ACK segment will be ignored; a corrupt byte or two in am image file will probably not be noticed, and no one would be surprised if a document I sent them had a spelling error.

Then again sometimes the database is corrupt and applications do randomly fail. Are these the result of a corrupt TCP segment? I don't know, maybe. I started researching this because of a claim that the TCP stack allowed a corrupt segment to update a database. Unfortunately, there were no traces showing the segment and no analysis was done of the database record to see if the corrupt section of the record had the same checksum value as the uncorrupted original section. The best answer I could give the DBA was that it could have happened.

It seems to me that the level of paranoia should be based on the criticality of the data. E-mailing pictures of the kids to my mother is one thing, updating a medical database or making a financial transaction is something else entirely.

## How to protect your critical data during transfer

By definition the Ethernet and TCP stacks cannot protect your data from errors that are undetectable. You can however ensure that any corruption is still detected by you or your application.

If you are transferring files via FTP or some other protocol you can zip the file before transferring. A corrupted file will not unzip correctly. Depending on the file this may have the added benefit of reducing the file size, fewer bits means less probability of undetectable errors and a shorter transfer time. If you are transferring data in an application you can add a hash (MD5, SHA-1, or something similar) of the data as part of each application layer message that is being sent. This will add bits to the message and CPU processing time but you will be guaranteed that any data corruption will be detected by the receiving application.

-----------------------------
[1] Performance of Checksums and CRCs over Real Data" Stone and Partridge in IEEE/ACM Transactions on Networking volume 6 number 5 1998

---
1. 见书 P573
2. 原文链接：http://noahdavids.org/self_published/CRC_and_checksum.html
