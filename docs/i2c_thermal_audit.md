# LM75 Scan Over I2C on 82599 Chipset
Related patch: [LM75x scan over i2c](/patches/i2c_thermal_audit.patch)    
This attempt includes testing if there is lm75 near the ASIC on the PCB.    
LM75 Document used : [lm75x][lm75_doc]    
82599 Document used : [82599][82599]    

## Reason       
Due to lack of observability on NIC's thermal state;     
As is common practice among homelabbers, it's sometimes mandatory to pin-tape SMBus management & SMBus clock pins(PCIe B5 & B6) in some situations with consumer motherboards for various reasons(e.g; [thread from superuser website][superuser_example])     
So I wondered is there any LM75 sensor. Since we are not looking for inside the ASIC, 82599 won't help, and of course design of the NIC is not open source.      
So only option left is interrogating the I2C bus via bit-banging.
> Note that i2c-tools is checked first. ixgbe wasn't exposing it's I2C's to host kernel, as expected since we have software defined pins, and also it can cause deadlocks if there is more than one user for NIC I2C.      
## Implementation       
Since there is a lot of LM75 sensor in the market,we can't check any identifier. The only option left is doing a sanity check. 
As we can see on [lm75x][lm75_doc] part 3, this sensor can report between -55 to 125, while the 82599 Tcase max is 119( [82599 datasheet][82599] p 988 table 13-2).     
## Result     
No ACK received for addresses 0x48-0x4F, confirming lack of slave response.     
Since there is different PCB designs in the market; 
- Different results may be encountered on different PCB designs,
- PCB may include it, but it's not connected to I2C.
At first, I tried to only use pr_info to print a log if it found something, or to inform that it can't find anything. With this implementation,we learnt we don't have any LM75 chips. After this attempt, the result was already reached.      
After this attempt was unsuccessful, I used pr_info for every attempt(8 possible addresses) for root cause analysis(see [logs](/logs/i2c_thermal_audit.log)). In the helper function, we have 19 (no such device) and 22 (invalid argument) errors, and we got 5(**I/O err**) means we succeeded scanning I2C ports,but no sensor acknowledged(NACK).       
This result must not be related with SMBus pin-taping since we are performing bit-banging directly via the ASIC's software definable pins([82599 datasheet][82599] table 8-2,p 521), bypassing the physical PCIe SMBus pins (B5/B6). This confirms the sensor is physically missing from the PCB, not just isolated by pin-taping.
## Side effects observed      
- Latency at init :    
I already guessed I2C will cause latency, but not 15 seconds at init.     
With vanilla driver, it takes 2.218064 seconds to set SFP's up after first dmesg log.    
With implementation,it was increased to 17.455836 seconds. Totally 8 i2c port was scanned per SFP(2). Note that 16 I2C call can't take 17.4~ seconds,( 1.0875~ seconds per call)82599 datasheet includes SMBus and I2C specifications, which is measured in microseconds(table 11-18 on [82599 datasheet][82599],p 928) so I investigated;     
In ixgbe_read_i2c_byte_generic_int function (we used ixgbe_read_i2c_byte_generic, but generic_int has the core logic), we can see '**max_retry = 10**' and '**msleep(100)**', so in case of a fail(which we are investigating on) this sleep simply causes 1 secs. Also, if firmware was busy, we may also waiting that(which is probably, since we are running it in probe func). This explains 1~ secs per address.     
- SFP flaps related to I2C scan :      
SFP is flapping because they are using the same I2C interface(since there is only one I2C interface : ([82599 datasheet][82599], Table 8-2 Register Summary, p521. "0x00028 I2CCTL I2C Control Target RW PERST 547".)),and I2C is not fast enough to meet both of their requirements at the given time. Also, keep in mind that this doesn't effect the result of our I2C LM75x scan, considering the return code(I/O err) and 10 retries which is implemented in the helper we called(ixgbe_read_i2c_byte_generic).
This patch is only called at init and doesn't affect line-rate performance, anyways it was a experimental patch intended for a one-shot run.      
If we want to go more depth, using I2C 15 times without releasing Semaphore(manager of resources between firmware and software, see [82599 datasheet][82599] 10.5.4 Software and Firmware Synchronization, p 914.) is cousing SFP modules to think they are cannot communicating with software, so they are powering themselves off.      
if it contained running while network traffic, it could cause errors related to out of descriptors/ring buffers and/or related to SFP-I2C datapath, which caused link flaps at probe. Perf logs at the end of appendix are proving everything said on this part.
## Problems encountered in implementation      
In first implementation, I passed 0x48 directly to ixgbe_read_i2c_byte_generic,which caused to read 0x24, since I2C ifaces are 7 bit for data + 1 for read/write(0 for write, 1 for read). Bad part is, the logs are containing 0x48,since they are not bit-banging. the fix is simple,      
``      
-status = ixgbe_read_i2c_byte_generic(hw, LM75_TEMP_REG, addr,     
+status = ixgbe_read_i2c_byte_generic(hw, LM75_TEMP_REG, addr << 1,
``.      
Before and after this implementation, the results are not changed(may be changed), but first implementation is totally invalid since it's scanning unrelated addresses.     
### Appendix       
Time between probes of 2 address was **1.04~** secs: 
[  373.212773] ixgbe 0000:01:00.1: Probing 0x49; Status: -5, Data: 0x00     
[  374.252849] ixgbe 0000:01:00.1: Probing 0x4a; Status: -5, Data: 0x00     
Total init was increased **17.4~** seconds(Note that 2 SFP ports doubles it):    
without patch:      
[    7.705961] ixgbe: Intel(R) 10 Gigabit PCI Express Network Driver    
...    
[   10.017793] ixgbe 0000:01:00.1 eth2: NIC Link is Up 10 Gbps, Flow Control: RX/TX    
with patch:    
[   85.165729] ixgbe: Intel(R) 10 Gigabit PCI Express Network Driver      
...    
[  102.621565] ixgbe 0000:01:00.0 eth1: NIC Link is Up 10 Gbps, Flow Control: RX/TX   
Scanning port 1 is cousing other port to flap since they are both using the same I2C :     
[   89.071206] ixgbe 0000:01:00.0 eth1: detected SFP+: 3      
[   89.300243] ixgbe 0000:01:00.0 eth1: NIC Link is Up 10 Gbps,  Flow Control: RX/TX    
*Eth2 scan*       
[   97.559109] ixgbe 0000:01:00.0 eth1: NIC Link is Down      
[  102.911669] ixgbe 0000:01:00.0 eth1: NIC Link is Up 10 Gbps, Flow Control: RX/TX      
As we can see, the I2C usage is locking semaphore,which causes the SFP to think that host is dead and downs the link.      
> Raw log files aren't shared(perf etc.) because they are huge and contains normal behaviour nearly every line,so I only included the evidence parts.      
     
[superuser_example]: https://superuser.com/questions/958282/computer-wont-boot-with-intel-10gb-nic-installed
[lm75_doc]: https://www.ti.com/lit/ds/symlink/lm75b.pdf?ts=1769495801859&ref_url=https%253A%252F%252Fwww.google.com%252F
[82599]: https://www.intel.com/content/www/us/en/content-details/331520/intel-82599-10-gigabit-ethernet-controller-datasheet.html