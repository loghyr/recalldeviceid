---
title: Add CB_LAYOUTRECALL_DEVICE to NFSv4.2
abbrev: RECALL_DEVICE
docname: draft-haynes-nfsv4-recalldevice-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: Transport
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC2119:
  RFC4506:
  RFC7862:
  RFC7863:
  RFC8174:
  RFC8178:
  RFC8434:
  RFC8435:
  RFC8881:

informative:
  RFC1813:

--- abstract

The Parallel Network File System (pNFS) allows for the metadata
server to use CB_LAYOUTRECALL to recall a layout from a client
by file id or file system id or all. It also allows the server to
use CB_NOTIFY_DEVICEID to delete a deviceid. It does not provide
a mechanism for the metadata server to recall all layouts that
have a data file on a specific deviceid.  This document presents
an extension to RFC7862 to allow the server to recall layouts from
clients based on deviceid.

--- note_Note_to_Readers

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/browse/nfsv4/).
Working Group information can be found at
[](https://datatracker.ietf.org/wg/nfsv4/about/).

--- middle

# Introduction

In the Network File System version 4 (NFSv4) with a Parallel NFS
(pNFS) metadata server ({{RFC8881}}), there is no mechanism for the
metadata server to recall layouts from the client when a particular
deviceid (see Section 3.3.14 of {{RFC8881}}) either temporarily or
permanently is no longer available.

One use case is when the deviceids in a layout are separated by
power fault domains. Each layout might describe 3 different
devices, each contained in a different power fault domain. In
such a scenario, a single fault domain can have the power
removed and not cause the loss of access to the data.  However,
client I/O will be impacted as the client still has to perform
WRITEs (see Section 18.32 of {{RFC8881}}) to the unavailable device,
send LAYOUTERRORs (see Section 15.6 of {{RFC7862}}) to inform the
metadata server of NFS4ERR_NXIO (see Section 15.1.16.3 of
{{RFC8881}}).

If the metadata server had the means to recall layouts by deviceid,
a lot of this unnecessary traffic could be eliminated. Finally,
while the metadata server could recall layouts one by one, this
is again unnecessary traffic and can be offloaded to the client.

Besides the use case above, consider if the metadata server wants to
set the NOTIFY4_DEVICEID_DELETE in the CB_NOTIFY_DEVICEID callback
(see Section 20.12 of {{RFC8881}}). This flag cannot be set if a layout
is outstanding for a deviceid. While the metadata server can revoke
all such layouts, there is no way to know that the client has
acknowledged that revocation and hence is still not doing I/O
to other data files in the layout. The metadata server could fence
those layouts as well (see Section 12.5.5 of {{RFC8881}}), but that
can be an expensive operation.

Using the process detailed in {{RFC8178}}, the revisions in this
document become an extension of NFSv4.2 {{RFC7862}}. They are built
on top of the external data representation (XDR) {{RFC4506}} generated
from {{RFC7863}}.

## Do we need {{RFC8435}}?
{:removeInRFC="true"}

The authors have tried to introduce this new functionality outside
of a particular pNFS Layout Type. Does that work?

# Requirements Language

{::boilerplate bcp14-tagged}

# Extension to Operation 5: CB_LAYOUTRECALL - Recall Layout from Client {#op_CB_LAYOUTRECALL}

The original union layoutrecall4 (see Section 20.3.1 of {{RFC8881}}) is:

~~~ xdr
enum layoutrecall_type4 {
        LAYOUTRECALL4_FILE = LAYOUT4_RET_REC_FILE,
        LAYOUTRECALL4_FSID = LAYOUT4_RET_REC_FSID,
        LAYOUTRECALL4_ALL  = LAYOUT4_RET_REC_ALL
};

union layoutrecall4 switch(layoutrecall_type4 lor_recalltype) {
   case LAYOUTRECALL4_FILE:
           layoutrecall_file4 lor_layout;
   case LAYOUTRECALL4_FSID:
           fsid4              lor_fsid;
   case LAYOUTRECALL4_ALL:
           void;
   };
~~~

The proposed extension is:

~~~ xdr
///    const LAYOUT4_RET_REC_ALL       = 4;
///
///    enum layoutrecall_type4 {
///           LAYOUTRECALL4_FILE = LAYOUT4_RET_REC_FILE,
///           LAYOUTRECALL4_FSID = LAYOUT4_RET_REC_FSID,
///           LAYOUTRECALL4_ALL  = LAYOUT4_RET_REC_ALL,
///           LAYOUTRECALL4_DEVICEID = LAYOUTRECALL4_RET_REC_DEVICEID
///   };
///
/// union layoutrecall4 switch(layoutrecall_type4 lor_recalltype) {
///   case LAYOUTRECALL4_FILE:
///           layoutrecall_file4 lor_layout;
///   case LAYOUTRECALL4_FSID:
///           fsid4              lor_fsid;
///   case LAYOUTRECALL4_DEVICEID:
///           deviceid4          lor_deviceid;
///   case LAYOUTRECALL4_ALL:
///           void;
///   };
~~~

With this minimal change, all of the semantics of CB_LAYOUTRECALL
(see Section 20.3 of {{RFC8881}}) remain the same, i.e., the client
and server are aware of how CB_LAYOUTRECALL interacts with each
other. The one issue to be investigated is what happens if an
NFSv4.2 client sees a LAYOUTRECALL4_DEVICEID in a CB_LAYOUTRECALL.
They SHOULD return NFS4ERR_UNION_NOTSUPP, but the implementations
might not be compliant with {{RFC8178}}. As such, a survey should
be conducted of the major implementations.

Finally, when the client does handle a LAYOUTRECALL4_DEVICEID in a
CB_LAYOUTRECALL, it MUST return all layouts which have a given
deviceid.  The server can determine that the client no longer has
any layouts with the given deviceid once the client replies with
NFS4ERR_NOMATCHING_LAYOUT.

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the new open flags for delegating the file
to the client.  The XDR description is presented in a manner that
facilitates easy extraction into a ready-to-compile format. To
extract the machine-readable XDR description, use the following
shell script:

~~~ shell
<CODE BEGINS>
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
<CODE ENDS>
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
<CODE BEGINS>
sh extract.sh < spec.txt > layout_wcc.x
<CODE ENDS>
~~~

This script removes leading blank spaces and the sentinel sequence '///'
from each line. XDR descriptions with the sentinel sequence are embedded
throughout the document.

Note that the XDR code contained in this document depends on types from
the NFSv4.2 nfs4_prot.x file (generated from {{RFC7863}}).  This includes
both nfs types that end with a 4, such as offset4, length4, etc., as
well as more generic types such as uint32_t and uint64_t.

While the XDR can be appended to that from {{RFC7863}}, the various code
snippets belong in their respective areas of that XDR.

# Security Considerations

There are no new security considerations beyond those in {{RFC7862}}.

# IANA Considerations

IANA should use the current document (RFC-TBD) as the reference for
the new entries.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust and Paul Saab were involved in the initial
requirements for this functionality.

Brian Pawlowski and Gorry Fairhurst helped guide
this process.
