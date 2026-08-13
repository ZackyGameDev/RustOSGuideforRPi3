# Writing an Operating System in Rust

A beginner-friendly, step-by-step guide to building a bare-metal
operating system from scratch in Rust for the Raspberry Pi 3 Model B+.

## Introduction

This book documents the development of AtOS, a small operating system written from scratch in Rust for the Raspberry Pi 3 Model B+. It is both
a practical guide to operating-system development and a record of the decisions and implementations made while building AtOS.

The guide covers bare-metal Rust and AArch64 development on the Raspberry Pi 3 Model B+. Although things learned in this guide
will definitely be useful in other Raspberry Pi models, or any other standard computer hardware.

Naturally, as I discover more information while developing AtOS, I may have to amend previous chapters. The guide is written primarily for
beginners, so concepts are introduced progressively rather than assuming extensive operating-system knowledge.

## Structure

The structure is extremely simple. The chapters follow a journey like chronological order. Instead of explaining one aspect at a time, they are meant to be followed through in an episodic format. Meant to cover implementations the same order as the order in which a real developer would tackle them in his project. However this continuity starts from Chapter 1. Chapter 0 is more akin to rough a prologue chapter for people who have absolutely minimal knowledge about the subject.  

## Contribution

This guide is still incomplete so the final proof reading and correction is yet to be done. However, no pull requests will ever be accepted, not right now, neither after this guide is complete. Issues however can be freely opened about mistakes, suggestions, recommendations, etc. 

## AtOS

My operating system which this entire guide is technically a documentation of, is named "AtOS". All the source code is pubicly available in the below linked repository. Chapters may also link to its old snapshots for reference. While this book does not accept pull requests, AtOS accepts any reasonable contributions to it. However AtOS is not meant to be a large scope operating system. It is only meant to serve as the foundation for this guide, so adding complicated features which are not covered by this guide will not be accepted.

## Acknowledgements

In chapter 5, the code related to input handling was contributed to the original AtOS repository by GitHub contributor Tanishq Daiya ([@tanishqdaiya](https://github.com/tanishqdaiya))

## Links 

- Free online book version: [zackygamedev.github.io/RustOSGuideforRPi3](https://zackygamedev.github.io/RustOSGuideforRPi3/)
- Source Repository: [github.com/ZackyGameDev/RustOSGuideforRPi3](https://github.com/ZackyGameDev/RustOSGuideforRPi3)
- AtOS source code: [github.com/ZackyGameDev/AtOS/](https://github.com/ZackyGameDev/AtOS/)
