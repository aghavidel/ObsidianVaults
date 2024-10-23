Here, a summary of the thesis chapter in "Towards Correct by Design Scalable Control Plane" paper shall be provided. This is supposed to be a primer for what would become a separate project at NSL, but at the moment it has been put on hold.

## Introduction

The goal is to design scalable control planes for SDNs, leveraging techniques from software engineering and formal verification. The [[Microservice]] architecture is yet to be used fully in SDN controller (probably with the exception of controllers such as [[Orion]], which are still new), it creates many new challenges and also alleviates previous ones.

Microservices rely on a dedicated network to route RPCs between loosely connected modules that make up an entire service.