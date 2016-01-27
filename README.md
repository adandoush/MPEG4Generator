Applications
------------

Mpeg- Model Description
*****************

The video generator module goal is to provide a flexible configurable packet generator of MPEG-4-like UDP traffic. The source code for the new module so-called (MpegPktGenClient) lives in the directory ``src/applications``. A helper module (MpegPktGenClientHelper) is also implemented to facilitate it's use. We can change the video streaming traffic shape, i.e. the GOP, by simply changing the key parameters values explained hereafter. The Video generator client can work with any udp based server like the already implemented in NS-3 UdpServer. An example of usage, expected results, and an analysis of the output trace are presented. 

Attributes
==========

The Mpeg packet generator module is a configurable application through the following key attributes (a default value is set to every attribute).

* GopLength: number of frames (I + P or B) per Burst period. By default, its value is set to (15).

* ImageRate: number of frames per Second fps created by the Camera. By default, its value is set to (12 fps).

* IframeSize: size of the I-frames in the GOP in kbits. By default, its value is set to (1200 kiloBits).

* AvBitRate: average Bit rate in mbps. By default, its value is set to (2 mbps).

* PeakRate: peak Bit rate at which we send the I-Frames in mbps. By default, its value is set to (10 mbps).

* MaxPacketSize: the maximum size of a packet in Bytes without the headers. By default, it takes 1468 Bytes including 12 Bytes for the ``App_header`` , additional 8 Bytes for ``udp_header`` and  20 for the ``IPv4_header`` and 14 bytes for ``Eth_header`` [Macs, MacD, Type]) + 4 Bytes for ``Frame Chk Seq`` will be added to the payload. This yields to a frame of size equals to 1514Bytes and this is matching the video trace provided by ALSTOM's team.

* VideoFilename: name of the text file to save the video calculated parameters to. By default, it save to mpeg-4-auto-gen.dat.

* The RemotePort and RemoteAdress that the application connect to.

The application calculate first the other required parameters of the video generator and save the characteristics in the text file pointed by the user or in the mpeg-4-auto-gen.dat (if empty name by the user is provided) as we will explain in the next section.

Design and calculation
======================
Our objective is to simulate the packet generation events for the frames of a GOP and to repeate the process until the stop time of the application. To that end, we have to calculate the number of packets for each frame and the inter packet durations. Then, we can scheduler the sending events.

The basic calculation is done as follow:
1. calculate the ``m_burstPeriod`` as the value of (double)m_gopLength/(double)m_imageRate;

2. m_burstDuration = (double)m_iFrameSize*1000/(double)(m_peakBitRate*1000000);

3. The amount of data in a Gop is computed as::
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
=====       =========       =============    ==============
index	      frameType	      timeToSend ms    frame_I_size B 
=====       =========       =============    ==============
1	              I             		  0	            	150000 
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
=====       =========       =============    ==============      

Verification:: The Calculated bitRate in each GoP as the data size during the GOP over the Gop Period is:2.5Mbits/1250MSec = 2Mbps

8. Our next step is to tranfer the characteristics into a packets generator while respecting the average Bit rate and the packet size key attributes (e.g. m_avBitRate=2 mbps and m_maxPacketSize=1468 Bytes).

Usage 
*****

The module usage is extremely simple. The helper will take care of about everything.

The typical use is::

  // Flow monitor
  Ptr<FlowMonitor> flowMonitor;
  FlowMonitorHelper flowHelper;
  flowMonitor = flowHelper.InstallAll();

  Simulator::Stop (Seconds(stop_time));
  Simulator::Run ();

  flowMonitor->SerializeToXmlFile("NameOfFile.xml", true, true);

the ``SerializeToXmlFile ()`` function 2nd and 3rd parameters are used respectively to
activate/deactivate the histograms and the per-probe detailed stats.

Other possible alternatives can be found in the Doxygen documentation.


Helpers
=======

The helper API follows the pattern usage of normal helpers.
Through the helper you can install the monitor in the nodes, set the monitor attributes, and 
print the statistics.

One important thing is: the :cpp:class:`ns3::FlowMonitorHelper` must be instantiated only
once in the main. 

Example and output analysis
===========================
The examples are located in `src/examples`.

The main model output is an XML formatted report about flow statistics. An example is::


The output was generated by a TCP flow from 10.1.3.1 to 10.1.2.2.

It is worth noticing that the index 2 probe is reporting more packets and more bytes than the other probles. 
That's a perfectly normal behaviour, as packets are fragmented at IP level in that node.

It should also be observed that the receiving node's probe (index 4) doesn't count the fragments, as the 
reassembly is done before the probing point.


Validation
**********

The paper in the references contains a full description of the module validation against
a test network.
