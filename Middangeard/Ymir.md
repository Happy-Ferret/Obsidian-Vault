# Ymir

## Middangeard archive format

Ymir is a simple, extensive archive format.
A fork of [asar](https://github.com/electron/asar), it works like `tar` that concatenates all files together without compression, while having random access support.

### v0
- Flat archive
	- No internal folders
	- All contents compiled into root of `.y0` file 
- Only assets, no executables or scripts