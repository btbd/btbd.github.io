
# Mirror's Edge

## Faith

Since Faith's body is visible and acts as its own actor with properties, Faith is controlled by two actors: her body and her camera.

### Faith's Body

The easiest way to get a valid pointer to the base address of Faith's body actor on any game version is to first search for the following assembly code in the range of `MirrorsEdge.exe`:
```asm
mov DWORD PTR 0x00000000,ecx ; 0x00000000 represents the static offset
```
Which is made up of the following bytes (in hex): `89 0D 00 00 00 00`.

However, there are multiple `mov DWORD PTR` instructions, so the next two instruction must also be searched for:
```asm
mov DWORD PTR 0x00000000,ecx ; 89 0D ?? ?? ?? ??
mov ecx, 0x00000000 ; B9 ?? ?? ?? ??
jmp r/m32 ; FF ??
```

Thus, the final byte pattern is: `89 0D ?? ?? ?? ?? B9 ?? ?? ?? ?? FF`.
Note: the final `??` byte is excluded because it is not necessary to search for.

After retrieving the static offset, the following pointer offsets are applied:
`static_offset, CC, 4A4, 214, 0`


#### Pseudocode
```cpp
DWORD body_base = FindPattern(EXE.modBaseAddr, // Base
                              EXE.modBaseSize, // Search length
                              "\x89\x0D\x00\x00\x00\x00\xB9\x00\x00\x00\x00\xFF", // Bytes
                              "xx????x????x"); // Mask
							  
/* The first 2 bytes are the opcode and the r/32 byte, 
 * so the last 4 bytes of the instruction (the static offset) are needed */
body_base = ReadInt((void *)(body_base + 2));

body_base = GetPointer(body_base, 0xCC, 0x4A4, 0x214, 0x00);
```
#### Structure

These are the currently known offsets/structure values for Faith's body actor (technically applicable to any actor):

Note: X and Y are on the horizontal axis, and Z is on the vertical axis.

| Offset (Hex) | Type         	   | Description |
|:-------------|:------------------|:------------|
| 68 		   | Byte			   | State |
| 78, C 	   | Float		       | Idle Animation Delay |
| 78, 10       | Float		       | Idle Animation Timer |
| 7C		   | DWORD		       | Animation Object Count |
| E4, 20C      | Float			   | Friction |
| E8 		   | Float			   | Position X |
| EC		   | Float			   | Position Y |
| F0 		   | Float			   | Position Z |
| F4 		   | Float			   | Rotation X |
| F8		   | Float			   | Rotation Y |
| FC 		   | Float			   | Rotation Z |
| 100 		   | Float			   | Velocity X |
| 104		   | Float			   | Velocity Y |
| 108 		   | Float			   | Velocity Z |
| 10C		   | Float			   | Add Velocity X |
| 110 		   | Float			   | Add Velocity Y |
| 114		   | Float			   | Add Velocity Z (Wallrun) |
| 154		   | Float			   | Proportional Scale (XYZ) |
| 158		   | Float			   | Scale X |
| 15C		   | Float			   | Scale Y |
| 160		   | Float			   | Scale Z |
| 264		   | Float			   | Max Horizontal Velocity (Ground) |
| 274		   | Float			   | Speed Gain Constant |
| 2A0		   | QWORD		       | Wallrun State |
| 2A8		   | Float	           | Wallrun |
| 2B8		   | DWORD			   | Health |
| 2BC		   | DWORD			   | Max Health |
| 2E0 		   | Float			   | Last Ground X |
| 2E4		   | Float			   | Last Ground Y |
| 2E8 		   | Float			   | Last Ground Z |
| 4CC 		   | Float			   | Prediction Vector X |
| 4D0		   | Float			   | Prediction Vector Y |
| 4D4 		   | Float			   | Prediction Vector Z |
| 4F4		   | Byte			   | Hand State |
| 4FE		   | Byte			   | Movement State |
| 503 		   | Byte			   | Walking State |
| 505		   | Byte 			   | Action State |
| 5CC 		   | Float			   | Offset X |
| 5D0		   | Float			   | Offset Y |
| 5D4 		   | Float			   | Offset Z |
| 72C		   | Float		       | Fall Z |

### Faith's Camera