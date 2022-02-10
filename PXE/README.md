# PXE Booting

In general, the 2020's seem to want to suggest that deploying images is a much better option than PXE booting. This is kind of true, but realistically, the answer relies more on what you intend to do with the image you build.

If you're looking to deploy to ***"The Cloud"***, you're probably looking to deploy multiple images and then clean up afterwards. If you're not looking to do this, you're doing ***"The Cloud"*** wrong. ***"The Cloud"*** is not really designed for you to put your single, hand-crafted, server image into circulation. It's designed for battery farming. Load your shed full of all the cattle images you can handle and then shoot anything that looks sick. Refer to the [**My Thoughts**](../My Thoughts.md) document in the root for more.

If you're looking to deploy more curated servers, PXE booting, and indeed, historical setup processes, are a better approach. If you're deploying a handful of servers for something, images are almost more trouble than they're worth. And, original processes actually make more sense too. Redeploying being better than updating is a hard argument to win at 5 servers. Images are an economy of scale process. If you're deploying more than 5 servers, images and redeploying. If you're only deploying 2, PXE booting, build and update.

# The Process

PXE booting is an ancient relic of bygone eras, that is the way it is because it's **perfectly functional**. There are very few reasons to redesign the wheel, and there are just as few reasons to redesign PXE booting. That said, extending on the concept of PXE booting is an amazing idea, and has been done incredibly well by [iPXE](https://ipxe.org).

What follows is high praise of iPXE and the way it does things, with a dash of the small details and some examples of why you don't want to try and PXE boot Windows without iPXE. If you've already got your PXE build down, this is probably not useful to you.

Initially, PXE booting is a simple process. On your preferred system, install a TFTP server and then set up your DHCP server to set the option "Next Server" to said TFTP server. However, there are a number of caveats to this that are widely documented. Because of the utility of it in the rest of the repository, I'll be concentrating on Linux as my OS of choice. It's perfectly valid for BSD and Windows to be used, but Linux is my current go-to.

## Caveat

[PXE Chainloading Loop](https://ipxe.org/howto/dhcpd#pxe_chainloading)

iPXE is loaded from your TFTP server and it then becomes your boot environment. It's very cool, had tonnes of options and is AMAZINGLY good at what it's built for. Unfortunately, it has a small problem with the standard process. See, after iPXE loads, it will attempt to do the PXE process for you, calling DHCP to get an IP Address, then following the DHCP servers instruction to go to the "Next Server" and boot the specified "Boot File". This will result in a loop, as it attempts to PXE boot itself and then loops for another attempt.

Given a DHCP Server configuration similar to this:

```conf
subnet 192.168.0.0 netmask 255.255.255.0 {
  filename "ipxe.efi";
  next-server 192.168.0.1;
  range 192.168.0.10 192.168.0.250;
}
```

The process looks like this:

```
Start PC

--- DHCP Starts ---

BIOS asks for a DHCP server (DHCP Discover)
DHCP Server responds (DHCP Offer)
BIOS responds with an acceptance (DHCP Request)
DHCP Server marks IP as assigned (DHCP Acknowledge)
DHCP Server responds with acceptance of request and DHCP options including Next Server ("192.168.0.1") and File Name ("ipxe.efi")

--- DHCP Ends ---
--- PXE Starts ---

BIOS attempts to load given File Name ("ipxe.efi") from given server ("192.168.0.1") using TFTP
BIOS executes the "ipxe.efi" file

--- PXE Ends ---
--- iPXE Starts ----

iPXE asks for a DHCP server (DCHP Discover)
DHCP Server responds (DHCP Offer)
iPXE responds with an acceptance (DHCP Request)
DHCP Server remarks IP as assigned (DCHP Acknowledge)
DHCP Server responds with acceptance of request and DHCP options including Next Server ("192.168.0.1") and File Name ("ipxe.efi")
iPXE attempts to load given File Name ("ipxe.efi") from given server ("192.168.0.1") using it's TFTP
iPXE executes the "ipxe.efi" file

--- iPXE Loops ---

```

This is funny, but also annoying. Luckily, there are a number of fixes. Being that we're planning to use a Linux Server as our DHCP Server (probably), we can use ISC DHCP and simply add some basic logic to the File Name options:

```conf
subnet 192.168.0.0 netmask 255.255.255.0 {
  if exists user-class and option user-class = "iPXE" {
    filename "bootfile.efi";
  } else {
    filename "ipxe.efi";
  }
  next-server 192.168.0.1;
  range 192.168.0.10 192.168.0.250;
}
```

This changes the flow very simply, making it similar to: 

```
Start PC

--- DHCP Starts ---

BIOS asks for a DHCP server (DHCP Discover)
DHCP Server responds (DHCP Offer)
BIOS responds with an acceptance (DHCP Request)
DHCP Server marks IP as assigned (DHCP Acknowledge)
DHCP server checks logic, notices that the "user-class" is not equal to "iPXE"
DHCP Server responds with acceptance of request and DHCP options including Next Server ("192.168.0.1") and File Name ("ipxe.efi")

--- DHCP Ends ---
--- PXE Starts ---

BIOS attempts to load given File Name ("ipxe.efi") from given server ("192.168.0.1") using TFTP
BIOS executes the "ipxe.efi" file

--- PXE Ends ---
--- iPXE Starts ----

iPXE asks for a DHCP server (DCHP Discover)
DHCP Server responds (DHCP Offer)
iPXE responds with an acceptance (DHCP Request)
DHCP Server remarks IP as assigned (DCHP Acknowledge)
DHCP server checks logic, notices that the "user-class" is set to "iPXE"
DHCP Server responds with acceptance of request and DHCP options including Next Server ("192.168.0.1") and File Name ("bootfile.efi")
iPXE attempts to load given File Name ("bootfile.efi") from given server ("192.168.0.1") using its TFTP
iPXE executes the "bootfile.efi" file

--- iPXE Ends ---

```

Solving the issue and allowing us to get on with the program.

## Booting

As described in the caveat, the process is ridiculously simple. Given a Linux server doing DHCP using the ISC DHCP server, the settings are as follows:

```conf
subnet 192.168.0.0 netmask 255.255.255.0 {
  if exists user-class and option user-class = "iPXE" {
    filename "bootfile.efi";
  } else {
    filename "ipxe.efi";
  }
  next-server 192.168.0.1;
  range 192.168.0.10 192.168.0.250;
}
```

Substituting your subnet of course.