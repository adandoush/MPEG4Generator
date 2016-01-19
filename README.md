# MPEG4Generator
Video generator for NS-3
A configurable generator of MPEG-4-like UDP based traffics
It is implemented as a separate module so-called (MpegPktGenClient). A helper class/module (MpegPktGenClientHelper) at the application level is also implemented to facilitate it's use. This Video generator client can work with any udp based server like the already implemented in NS-3 UdpServer and its helper UdpServerHelper as we will show through an example here after.
The Mpeg packet generator module is a configurable application level protocol through the following key attributes in the scenario script (a default value is set to every attribute).
1. GopLength: number of frames (I + P or B) per Burst period, by default, its value is set to (15)

2. ImageRate: number of frames per Second fps created by the Camera, by default, its value is set to (12 fps)

3. IframeSize: size of the I-frames in the GOP in kbits, by default, its value is set to (1200 kiloBits)

4. AvBitRate: average Bit rate in mbps, by default, its value is set to (2 mbps)

5. PeakRate:  peak Bit rate at which we send the I-Frames in mbps", by default, its value is set to (10 mbps)

6. MaxPacketSize: the maximum size of a packet in Bytes without the headers, by default, it takes 1468 Bytes including 12 Bytes for the App_header, additional 8Bytes for udp_header and  20 for the IPv4_header and 14 bytes for  Eth_header [Macs, MacD, Type]) + 4 Bytes for “Frame Chk Seq” will be added to the payload. This yields to a frame of size equals to 1514Bytes and this is matching the video trace provided by ALSTOM's team.

7. VideoFilename: name of the text file to save the video calculated parameters to. By default, it save to mpeg-4-auto-gen.dat.


8. The RemotePort and RemoteAdress that the application connect to.

The application calculate first the other required parameters of the video generator and save the characteristics in the text file pointed by the user or in the mpeg-4-auto-gen.dat (if empty name by the user is provided).

The basic calculation is done as follow:
1. calculate the m_burstPeriod as the value of (double)m_gopLength/(double)m_imageRate;
2. m_burstDuration = (double)m_iFrameSize*1000/(double)(m_peakBitRate*1000000);
3. The amount of data in a Gop is computed as :
avDataSizeInGop = m_avBitRate * m_burstPeriod;//in  mb/s * s = mbit

4. We subtract the I-Frame-Size from the total amount of data in a GOP to get the summation of the P-Frames sizes
sum_frames_P_sizes = (avDataSizeInGop * 1000000 - m_iFrameSize *1000)/8;// in Bytes
 
5. The size of each P-Frame and the bitRate at which we send them to respecte the avBitRate are calculated as follow:
m_pFrameSize = sum_frames_P_sizes / (m_gopLength - 1);// in bytes
m_pFRate = sum_frames_P_sizes * 8 / (m_burstPeriod-m_burstDuration);//bps

6. Given the m_burstDuration (duration of the I-Frame in a GOP), we calculate the inter-P-Frame durations
interval_P_frames = (m_burstPeriod-m_burstDuration) * 1000 / (m_gopLength) ;// I   P P P P P P P P P P P P P P I   P ...

7. Then we can organize the Frames in each GOP. An example of typical GOP (with the default value of the key parameters) can be seen in the auto-generated video file:

The Generator parameters are: GOPLength=15	 Peak Rate=10mbps 	Av bit rate=2mbps	 Image Rate=12 fps	 I-Frame-Size=1200Kbits 	BurstPerdiod=1250ms	BurstDuration=120ms	Pkt Size=1468 
The generated GOP of the video is: 
index	    frameType	    timeToSend ms    frame_I_size B 
1		I		0		150000 
2		P		270.667		11607.1 
3		P		345.333		11607.1 
4		P		420.333		11607.1 
5		P		495.333		11607.1 
6		P		570.333		11607.1 
7		P		645.333		11607.1 
8		P		720.333		11607.1 
9		P		795.333		11607.1 
10		P		870.333		11607.1 
11		P		945.333		11607.1 
12		P		1020.33		11607.1 
13		P		1095.33		11607.1 
14		P		1170.33		11607.1 
15		P		1245.33		11607.1 

Verification:: The Calculated bitRate in each GoP as the data size during the GOP over the Gop Period is:2.5Mbits/1250MSec = 2Mbps

8. Our next step is to tranfer the characteristics into a packets generator while respecting the average Bit rate and the packet size key attributes (e.g. m_avBitRate=2 mbps and m_maxPacketSize=1468 Bytes).

Thus, we have to calculate the number of packets of each frame and then the inter-packet-interval between consequetive packets of I-frames and P-Frames by sending packets of frame I with m_peakBitRate and with m_pFRate for packets of frame P. In other words, the key parameter m_avBitRate for transmitting each GOP's data must still be verified. Therefore, the next time to send a packet is (m_maxPacketSize-App_header) / dataRate, where dataRate is either the m_PeakBitRate or the m_pFRate according to the frame type.
For examples,  the time between two packets of the I-Frame is 1456*8/(10*10^6) = 0.0011 sec and for the P frames =  1456*8/1.15044*10^6 = 0.010 sec. So, the time to send the first packet of the second frames for instance equals to the time to send the second frame, e.g. after 270.667 sec from the starting time (see the auto-generated file depicted above), while the time to send the second packet of the second frame equals to  270.667 +  0.0102 sec after the starting time of the application because the frame is of type P.
