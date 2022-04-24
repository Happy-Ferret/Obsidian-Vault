# Ymir

## Middangeard archive format

Ymir is a simple, extensive archive format.
A fork of [asar](https://github.com/electron/asar), it works like `tar` that concatenates all files together without compression, while having random access support.

### v0
- Flat archive
	- No internal folders aside of generated `v0` root folder
	- All contents compiled into `v0/` of `.y0` file 
- Only assets, no executables or scripts

