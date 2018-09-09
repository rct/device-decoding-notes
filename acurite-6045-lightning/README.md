# Acurite 06045M Lightning Detector Decoding Notes

See [rtl_433/src/devices/acurite.c](https://github.com/merbanan/rtl_433/blob/5b5f012b676b87a327f2bb083becb9882fcaf16f/src/devices/acurite.c#L355) for the current state of the rtl_433 code.

See also rtl_433_tests/acurite


## Description of the protocol from src/devices/acurite.c

```
 * Acurite 06045m Lightning Sensor decoding.
 *
 * Specs:
 * - lightning strike count
 * - estimated distance to front of storm, up to 25 miles / 40 km
 * - Temperature -40 to 158 F / -40 to 70 C
 * - Humidity 1 - 99% RH
 *
 * Status Information sent per 06047M/01021 display
 * - (RF) interference (preventing lightning detection)
 * - low battery
 *
 *
 * Message format
 * --------------
 * Somewhat similar to 592TXR and 5-n-1 weather stations
 * Same pulse characteristics. checksum, and parity checking on data bytes.
 *
 * 0   1   2   3   4   5   6   7   8
 * CI II  BB  HH  ST  TT  LL  DD? KK
 *
 * C = Channel
 * I = ID
 * B = Battery + Message type 0x2f
 * S = Status/Message type/Temperature MSB.
 * T = Temperature
 * D = Lightning distance and status bits?
 * L = Lightning strike count.
 * K = Checksum
 *
 * Byte 0 - channel/?/ID?
 * - 0xC0: channel (A: 0xC, B: 0x8, C: 00)
 * - 0x3F: most significant 6 bits of bit ID
 *    (14 bits, same as Acurite Tower sensor family)
 *
 * Byte 1 - ID all 8 bits, no parity.
 * - 0xFF = least significant 8 bits of ID
 *    Note that ID is just a number and that least/most is not
 *    externally meaningful.
 *
 * Byte 2 - Battery and Message type
 * - Bitmask PBMMMMMM
 * - 0x80 = Parity
 * - 0x40 = 1 is battery OK, 0 is battery low
 * - 0x3f = Message type is 0x2f to indicate 06045M lightning
 *
 * Byte 3 - Humidity
 * - 0x80 - even parity
 * - 0x7f - humidity
 *
 * Byte 4 - Status (2 bits) and Temperature MSB (5 bits)
 * - Bitmask PAUTTTTT  (P = Parity, A = Active,  U = unknown, T = Temperature)
 * - 0x80 - even parity
 * - 0x40 - Active Mode
 *    Transmitting every 8 seconds (lightning possibly detected)
 *    normal, off, transmits every 24 seconds
 * - 0x20 - TBD: always off?
 * - 0x1F - Temperature most significant 5 bits
 *
 * Byte 5 - Temperature LSB (7 bits, 8th is parity)
 * - 0x80 - even parity
 * - 0x7F - Temperature least significant 7 bits
 *
 * Byte 6 - Lightning Strike count (7 bits, 8th is parity)
 * - 0x80 - even parity
 * - 0x7F - strike count (wraps at 127)
 *    Stored in EEPROM (or something non-volatile)
 *
 * Byte 7 - Edge of Storm Distance Approximation
 * - Bits PSSDDDDD  (P = Parity, S = Status, D = Distance
 * - 0x80 - even parity
 * - 0x40 - USSB1 (unknown strike status bit) - (possible activity?)
 *    currently decoded into "ussb1" output field
 *    @todo needs understanding
 * - 0x20 - RFI (radio frequency interference)
 *    @todo needs cross-checking with light and/or console
 * - 0x1F - distance to edge of storm (theory)
 *    value 0x1f is possible invalid value indication (value at power up)
 *    @todo determine if miles, km, or something else
 *    Note: Distance sometimes goes to 0 right after strike counter increment.
 *          Status bits might indicate validity of distance.
 *
 * Byte 8 - checksum. 8 bits, no parity.
 *
 * Data fields:
 * - active (vs standby) whether the AS39335 is in active scanning mode
 *     will be transmitting evey 8 seconds instead of every 24.
 * - RFI detected - the AS3935 uses broad RFI for detection
 *     Somewhat correlates with the Yellow LED, but stays set longer
 *     Short periods of RFI on is normal
 *     long periods of RFI means interference, solid yellow, relocate sensor
 * - Strike count - count of detection events, 7 bits, non-volatile
 * - Distance to edge of storm - See AS3935 documentation.
 *     sensor will make a distance estimate with each strike event.
 *     Units unknown, data needed from people with Acurite consoles
 *     0x1f (31) is invalid/undefined value, consumers should check for this.
 * - USSB1 - Unknown Strike Status Bit
 *     May indicate validity of distance estimate, cleared after sensor beeps
 *     Might need to also correlate against RFI bit.
 * - exception - bits that were invariant for me have changed.
 *     save raw_msg for further examination.
 *
 * @todo - check parity on bytes 2 - 7
 *
 * Additional reverse engineering needed:
 * @todo - Get distance to front of storm to match display
 * @todo - figure out remaining status bits and how to report
 */
```

