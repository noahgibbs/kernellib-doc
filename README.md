# kernellib-doc

DGD is a raw, low-level language. You can think of it as being comparable to Java or C with the system libraries. Certain common tasks that a long-running DGD application will need simply aren't part of the system libraries, just as is the case with other languages.

The [Kernellib](https://github.com/ChatTheatre/kernellib), formerly called the Kernel MUDLib, provides a number of features that a long-running persistent application, such as a game, is likely to need. By providing these features in a well-known standard way, higher-level DGD library code can be written in a way that is likely to work across many different applications.

The Kernellib is no longer the most recent library of that kind. Dworkin (DGD's author) is currently working on an expanded version of DGD called Hydra, and has a Kernellib-like library called [Cloud Server](https://github.com/dworkin/cloud-server) that is meant to serve a similar purpose.

Despite aiming to provide a more uniform interface than DGD, the Kernellib tries to maintain a lot of flexibility. Where more than one approach is reasonable (e.g. object lifecycle management,) Kernellib tries to provide hooks and APIs rather than a full implementation.

## More About the Kernellib

[Dgd](https://ChatTheatre.github.io/lpc-doc/) has [some unusual characteristics](https://ChatTheatre.github.io/lpc-doc/dgd/unusual.html). It also provides enormous flexibility. Kernellib provides control over those unusual characteristics, and provides sensible defaults on top of that flexibility.

* [Features of Kernellib](./features.md)
* [Installing the Kernellib](./installing.md)

* [Kernellib Directory Structure](./directories.md)
* [Default Kernellib Commands](./commands.md)

* [Kernellib Permissions](./permissions.md)
* [Resource Management](./resources.md)
* [Object Management](./object_management.md)

* [Miscellaneous other documentation about the Kernellib](./miscellaneous.md)

## Older Documentation

The old Phantasmal web site had a good set of documentation about building on the Kernellib. It's not perfect, some of it is outdated, but there's a lot there.

[If you'd like to sift through it, here it is.](./phant/)

## Copyright &amp; Status

Unless otherwise noted, the contents of this repository as declared public domain using the Creative Commons "No Rights Reserved" [CC0 Declaration](https://creativecommons.org/share-your-work/public-domain/cc0/).

### CC0 Declarations

Under US law it is a bit more difficult to declare something public domain (see [article](https://www.techdirt.com/articles/20150123/15564629797/why-we-still-cant-really-put-anything-public-domain-why-that-needs-to-change.shtml)). Creative Commons offers a [CC0 tool](https://creativecommons.org/choose/zero/) that combined with GPG signed commits to this repository, helps with this issue.

The following parties have declared their contributions to this repository as public domain:

* Noah Gibbs

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license"
     href="http://creativecommons.org/publicdomain/zero/1.0/">
    <img src="http://i.creativecommons.org/p/zero/1.0/88x31.png" style="border-style: none;" alt="CC0" />
  </a>
  <br />
  To the extent possible under law,
  <a rel="dct:publisher"
     href="https://codefol.io">
    <span property="dct:title">Noah Gibbs</span></a>
  has waived all copyright and related or neighboring rights to
  <span property="dct:title">Skotos-Doc repository</span>.
This work is published from:
<span property="vcard:Country" datatype="dct:ISO3166"
      content="UK" about="https://github.com/ChatTheatre/eOS-Doc">
  United Kingdom</span>.
</p>

## Contributing

This repository is intended as a "working documentation", which means that we are actively seeking documentation and proposals for improvements of all kinds.

We require that before any first PR from a contributor to this repository can be accepted, either their first commit must be GPG signed and it must either a general update to this README.md with a CC0 disclaimer as above, or every commit must contain a comment that include a similar CC0 disclaimer, which can be obtained from [Creative Commons CC0](https://creativecommons.org/choose/zero/).

