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

The basic calculation is done as follow::

    1. calculate the ``m_burstPeriod`` as the value of (double)m_gopLength/(double)m_imageRate;
    2. m_burstDuration = (double)m_iFrameSize*1000/(double)(m_peakBitRate*1000000);
    3. The amount of data in a Gop is computed as 
    avDataSizeInGop = m_avBitRate * m_burstPeriod;//in  mb/s * s = mbit
    4. We subtract the I-Frame-Size from the total amount of data in a GOP to get the summation of the P-Frames sizes
    sum_frames_P_sizes = (avDataSizeInGop * 1000000 - m_iFrameSize *1000)/8;// in Bytes
    5. The size of each P-Frame and the bitRate at which we send them to respecte the avBitRate are calculated as follow
    m_pFrameSize = sum_frames_P_sizes / (m_gopLength - 1);// in bytes
    m_pFRate = sum_frames_P_sizes * 8 / (m_burstPeriod-m_burstDuration);//bps
    6. Given the m_burstDuration (duration of the I-Frame in a GOP), we calculate the inter-P-Frame durations
    interval_P_frames = (m_burstPeriod-m_burstDuration) * 1000 / (m_gopLength) ;// I   P P P P P P P P P P P P P P I   P ...
    7. Then we can organize the Frames in each GOP. An example of typical GOP (with the default value of the key parameters) can be found in the mpeg.dat file
    8. Our next step is to tranfer the characteristics into a packets generator while respecting the average Bit rate and the packet size key attributes (e.g. m_avBitRate=2 mbps and m_maxPacketSize=1468 Bytes).

Usage 
*****

The module usage is extremely simple. The helper will take care of about everything. Default values can be changed easily as shown in the code below.

The typical use is to creat a UDP server Application that is listing on a particular port, and then to creat the Mpeg Packet generator client application using the desired values of its attributed as follow::

    //
    // Create one udpServer applications on node one to receive the video packets from our generator.
    //
    UdpServerHelper server (port);
    ApplicationContainer apps = server.Install (n.Get (1));
    apps.Start (Seconds (1));
    apps.Stop (Seconds (100));
    //
    // Create one MpegPktGenClient application to send UDP datagrams from node zero to
    // node one.
    //
    uint32_t MaxPacketSize = 1460;
    uint16_t GopLength = 15;
    std::string fn = "mpeg.dat";
    MpegPktGenClientHelper client (serverAddress, port,"");
    client.SetAttribute ("MaxPacketSize", UintegerValue (MaxPacketSize));//def 1460 Bytes
    client.SetAttribute ("VideoFilename", StringValue (fn));
    client.SetAttribute("GopLength",UintegerValue (GopLength));//def 15
    client.SetAttribute("ImageRate",UintegerValue (12));//def 12
    client.SetAttribute("IframeSize",UintegerValue (1200));// def 1200 kbits
    client.SetAttribute("AvBitRate",DoubleValue (2.0));//def 2 mbps
    client.SetAttribute("PeakRate",DoubleValue (10.0));//def 10 mbps
    apps = client.Install (n.Get (0));
    apps.Start (Seconds (1.0));
    apps.Stop (Seconds (100.0));


A complete example
==================
The complete example is located in `examples/mpeg-gen-pkt-client`.


Validation through output analysis
**********************************

We provide in this section a module validation against a simple test network.
