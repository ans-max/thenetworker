+++ 
draft = true
date = 2023-06-05T15:38:16+05:30
title = "Golang and IP Addresses"
description = "Working with ip addresses in Golang"
slug = ""
authors = []
tags = ['golang','networking','programming']
categories = []
externalLink = ""
series = []
math = true
+++

**Introduction**

Working with IP addresses is a fundamental part of any network engineer's life. Most of us know the theory, and many of us can even do IPv4 subnetting calculations by hand. However, I was always curious about how exactly the calculations work mathematically in software. In my quest to learn Golang, I decided to investigate what constructs the language provides for working with IP addresses. This post will only deal with IPv4 addresses, unless otherwise stated.

This post assumes the reader has basic understanding of golang, including its contructs

As a project I started to code a ip subnetting tool with following specifications

input : ipaddress in a prefix format example : 192.168.0.34/24
output : given a valid input and tool should be able to print following information

1) ip network
2) ip subnet mask
3) ip host mask
4) first ip
5) last ip

constraints: we have to use the standard golang library only, above are the enough for the minimum viable product 

The basic logic behind ip addressing is explained quite nicely at https://www.baeldung.com/cs/get-ip-range-from-subnet-mask

In short an ipaddress is a 32 bit (64 for Ipv6) some of these 32 bits are used to identify the subnet and remaining bits are used to hosts in the subnet, Prefix length identifies the number of bits assigned for the network, 

An IP address by itself is meaningless unless it is accompanied by a subnet mask or prefix length. The subnet mask determines which bits of the IP address are used to identify the network and which bits are used to identify the host.

For example, the IP address 192.168.1.1 with a subnet mask of 255.255.255.0 represents the host 192.168.1.1 on the network 192.168.1.0.

The total host bits would be then be given by 

$$
host bits = 32 - prefix 
$$

**Project Setup**

Assuming the golang is already setup, from any location (preferably your src folder ) run the following commands on the terminal

```console
$mkdir -p ipaddr
$cd ipaddr
$go mod init
$touch main.go
```

This would generate a directory structure as shown, the output may however differ 

```console
ipaddr$ tree
.
├── go.mod
└── main.go

0 directories, 2 files
```

All code shown here will reside in the single file main.go


**Golang's net package** 

Golang's net package provides a unified interface for all networking operations in Go. IP addresses in Go are represented by the IP type, which is a slice of bytes. The subnet mask is represented by the IPMask type, which is also a slice of bytes.

The net package provides a number of functions for working with IP addresses and subnet masks. For example, the ParseIP() function can be used to parse an IP address from a string, and the Mask() function can be used to combine an IP address and a subnet mask to create an IPNet object.

Armed with above information lets start with writing some helper functions, incase there usecase is not clear now, i would suggest to jump back to each section once we use them finally in our core part of the code

first, lets import all thats needed

```go
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"strconv"
	"strings"
)

```

Now our first function, 

```go
// Takes a byte array as input,
// returns the dotted decimal format as string
func ipv4MaskString(m []byte) string {
	//Ipv4 mask has to a 4 byte entry
	if len(m) != 4 { 
		panic("ipv4Mask: len must be 4 bytes")
	}

	return fmt.Sprintf("%d.%d.%d.%d", m[0], m[1], m[2], m[3])
```
In above section the function **ipv4MaskString** returns the dotted decimal format of the subnetmask, 


```go
//Takes a subnet mask as net.IPMask as input
//Returns the hostmask as net.IPMask
func HostMask(mask net.IPMask) net.IPMask {
	n := len(mask)
	hostmask := make(net.IPMask, n)
	for i := 0; i < n; i++ {
		hostmask[i] = ^mask[i]
	}
	return hostmask
}
```

If we take a subnet mask and invert its bits that results in whats called a host mask, the function **HostMask** calculates the same from the subnet mask both subnet mask and host mask are represented by the same type net.IPMask 

```go
//takes the network id and mask as input in net.IP and net.IPMask form
//Returns the last available ip
func LastHost(ip net.IP, hostmask net.IPMask) net.IP {
	lastip := make(net.IP, len(ip))
	for i := 0; i < len(ip); i++ {
		lastip[i] = ip[i] | hostmask[i]
	}
	return lastip
}
```
if the network id is bitwise OR ed with hostmask it would give us the last ip available in the subnet the function **LastHost** does the same for us, 

Now comes the main function to use the above

```go

//Expected input in format 192.168.0.34/24

func main() {
	if len(os.Args) != 2 {
		log.Fatalf("Usage: %s dotted-ip-addr\n", os.Args[0])
	}
	//Extracting ip and prefix from the input
	dotAddr := strings.Split(os.Args[1], "/")[0]
	prefix := strings.Split(os.Args[1], "/")[1]
	intprefix, err := strconv.Atoi(prefix)
	// if the prefix is not correct 
	if err != nil {
		log.Fatalln("Bad Prefix")
	}
	//net.ParseIP parses a string and returns a net.IP
	addr := net.ParseIP(dotAddr)
	if addr == nil {
		log.Fatalln("nil Invalid Address")
	}
	if intprefix > 32 {
		log.Fatalln("Bad Prefix")
	}
	//convert the interger prefix to subnet mask
	mask := net.CIDRMask(intprefix, 32)
	//calculate the hostmask
	hostmask := HostMask(mask)
	//calculate the network using the mask see https://pkg.go.dev/net#IP.Mask
	network := addr.Mask(mask)
	networkWithPrefix := network.String() + "/" + prefix
	ones, bit := mask.Size()
	//calculate the last ip of the network
	last := LastHost(network, hostmask)
	fmt.Println("Network Is ", networkWithPrefix,
		"\nAddress is ", addr.String(),
		"\nLeading ones count is ", ones,
		"\nSubnetMask is ", ipv4MaskString(mask),
		"\nHostMask is ", ipv4MaskString(hostmask),
		"\nLast IP is ", last.String())
}
```

In the main function body the input (example 192.168.0.34/24) is validated and parsed in ip and prefix, after some validations we proceed further to convert the prefix to subnet mask

Once we have the subnet mask we can then proceed to calculate the remaining values and print them as shown


To run the code, just run as shown below

```go
:ipaddr$ go run main.go 192.168.0.34/24
Network Is  192.168.0.0/24 
Address is  192.168.0.34 
Default mask length is  32 
Leading ones count is  24 
SubnetMask is 255.255.255.0 
HostMask is  0.0.0.255 
Network is  192.168.0.0 
Last IP is  192.168.0.255*
:ipaddr$
``` 
