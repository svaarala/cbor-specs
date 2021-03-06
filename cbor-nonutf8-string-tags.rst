==========================
CBOR non-UTF-8 string tags
==========================

Overview
========

This document specifies CBOR [1] tags applied to a CBOR byte string
representing non-UTF-8 text in a variant of UTF-8::

  Tag: 272
  Data item: byte string
  Semantics: Non-UTF-8 CESU-8 string
  Point of contact: Sami Vaarala <sami.vaarala@iki.fi>
  Description of semantics: https://github.com/svaarala/cbor-specs/blob/master/cbor-nonutf8-string-tags.rst

  Tag: 273
  Data item: byte string
  Semantics: Non-UTF-8 WTF-8 string
  Point of contact: Sami Vaarala <sami.vaarala@iki.fi>
  Description of semantics: https://github.com/svaarala/cbor-specs/blob/master/cbor-nonutf8-string-tags.rst

  Tag: 274
  Data item: byte string
  Semantics: Non-UTF-8 MUTF-8 string
  Point of contact: Sami Vaarala <sami.vaarala@iki.fi>
  Description of semantics: https://github.com/svaarala/cbor-specs/blob/master/cbor-nonutf8-string-tags.rst

Tags are specified for the following common UTF-8 variants:

* CESU-8 ("Compatibility Encoding Scheme for UTF-16: 8-Bit (CESU-8)") [2]

* WTF-8 ("Wobbly Transformation Format - 8-bit") [3]

* MUTF-8 ("Modified UTF-8") [4]

Semantics
=========

When a string to be encoded is not valid UTF-8, but conforms to a variation
of UTF-8 (such as CESU-8 or WTF-8), it can be encoded as a CBOR byte string
tagged with one of the tags specified in this document.  This allows the
decoder to recover the non-UTF-8 string data without any loss of
information: the exact bytes are preserved, and the decoded value can be
treated as a string rather than arbitrary binary data.

Valid UTF-8 strings must be encoded as CBOR text strings, not tagged byte
strings, so that interoperability of valid UTF-8 strings is not affected.
This means that an encoder may need to UTF-8 validate an input string during
encoding.  The semantics of decoding a valid UTF-8 string encoded as a tagged
byte string is up to the decoder.

There is some overlap between the different UTF-8 variants.  In particular,
a certain byte string may decode to the same codepoint sequence in more than
one variant.  An encoder may choose any compatible tag in such cases, see
more discussion below.

Example
=======

The following valid ECMAScript string cannot be encoded as UTF-8 and is thus
not a viable CBOR text string::

  var str = 'foo\ud800';

It can be encoded as a CBOR byte string with CESU-8 encoding, and the tag
272 (0x110) applied::

  d9 01 10
    46 66 6f 6f ed a0 80

If the decoding entity supports the tag, it can recover the original ECMAScript
string without loss of information.  If it doesn't support the tag, it can
still recover the byte string.

Rationale
=========

Some programming languages and environments deal with strings outside of strict
UTF-8.  For example, ECMAScript strings allow all 16-bit code points with no
restrictions.  Unpaired surrogates and invalid surrogate pairs are explicitly
allowed and used even in practical applications.

As a result, some of the allowed ECMAScript strings cannot be encoded in UTF-8
without losing information.

For example, this is a valid ECMAScript string::

  var str = 'foo\ud800';

The string cannot be UTF-8 encoded because UTF-8 does not allow codepoints in
the surrogate pair range.  The string could be modified to be valid UTF-8 e.g.
by replacing the unpaired surrogate with a U+FFFD replacement character, with
some loss of information.

The string can be CESU-8 encoded without losing information as::

  66 6f 6f ed a0 80

One could encode such a string as a CBOR byte string::

  46 66 6f 6f ed a0 80

However, a receiver of the CBOR encoded text would have no way of knowing,
at least without context, whether the data was intended to be pure binary
data or some form of text to be recovered.  For example, an ECMAScript CBOR
decoder would usually decode the byte string either as an ArrayBuffer or an
Uint8Array instead of a string.

One could use tag 27 (generic-object) [5] and some appropriate "typename"
(the value "text/cesu8" below is purely an example)::

  27([ "text/cesu8", <binary string data> ])

  d8 1b 82
    6a 74 65 78 74 2f 63 65 73 75 38
    46 66 6f 6f ed a0 80

This is workable but has two major downsides:

1. There's considerable overhead.  In the example above, the raw byte string
   uses 7 bytes while the tagging uses 14 bytes (+200%).

2. If the decoder doesn't understand tag 27 (generic-object), the decoded
   result will be a CBOR Array which is generally worse than decoding the
   input as a byte string.  This is especially undesirable when the string
   is a key of a CBOR Map.

The tags specified in this document allow a better encoding::

  d9 01 10
    46 66 6f 6f ed a0 80

This is much better than when using tag 27 (generic-object):

1. There's less overhead.  In this example, the raw byte string uses
   7 bytes while the tagging uses 3 bytes (about +43%), a major
   improvement.

2. A decoder which does not support the tag will still yield a byte string
   which is no worse than when using a bare byte string.

The tags specified in this document must not be applied to valid UTF-8
strings, only strings that cannot be encoded as valid CBOR text strings.
This has two major benefits:

1. Valid UTF-8 strings, which dominate applications, remain plain CBOR
   and require no support for custom tags, maximizing interoperability.

2. The encoding of a valid UTF-8 string as a CBOR text string is shorter
   than as a tagged CBOR byte string.

The downside of this requirement is that an encoder may need to check
the UTF-8 validity of an input string before deciding on the appropriate
encoding, which has a minor performance impact.

Overlap between UTF-8 variants
==============================

There is some overlap between the UTF-8 variants; for example:

* All CESU-8 strings containing no valid paired surrogates are valid WTF-8,
  and have the same meaning.

* All WTF-8 strings containing no non-BMP codepoints (> U+FFFF) are valid CESU-8,
  and have the same meaning.

* All CESU-8 strings containing no NUL (0x00) are valid MUTF-8 and vice versa,
  and have the same meaning.

* All WTF-8 strings containing no NUL (0x00) or non-BMP codepoints (> U+FFFF)
  are valid CESU-8, WTF-8, and MUTF-8, and have the same meaning.

* A WTF-8 string containing a non-BMP codepoint (> U+FFFF) is invalid CESU-8
  and MUTF-8.

* A MUTF-8 string containing a NUL (0xc0 0x80) is invalid CESU-8 and WTF-8.

The table below provides some examples of the overlap, providing the decoded
codepoint sequence for each variant:

+----------------------------+----------------+----------------+----------------+
| Byte sequence              | CESU-8         | WTF-8          | MUTF-8         |
+============================+================+================+================+
| 66 6f 6f ed a0 80          | 66 6f 6f d800  | 66 6f 6f d800  | 66 6f 6f d800  |
+----------------------------+----------------+----------------+----------------+
| 66 6f 00 ed a0 80          | 66 6f 00 d800  | 66 6f 00 d800  | invalid        |
|                            |                |                | (one-byte NUL) |
+----------------------------+----------------+----------------+----------------+
| ed a0 bd ed b2 ac          | d83d dcac      | invalid        | d83d dcac      |
|                            |                | (paired)       |                |
+----------------------------+----------------+----------------+----------------+
| ed a0 bd ed b2 ac 00       | d83d dcac 00   | invalid        | invalid        |
|                            |                | (paired)       | (one-byte NUL) |
+----------------------------+----------------+----------------+----------------+
| f0 9f 92 ac                | invalid        | 1f4a9          | invalid        |
|                            | (non-BMP)      |                | (non-BMP)      |
+----------------------------+----------------+----------------+----------------+
| 66 6f 6f c0 80             | invalid        | invalid        | 66 6f 6f 00    |
|                            | (two-byte NUL) | (two-byte NUL) |                |
+----------------------------+----------------+----------------+----------------+
| ff                         | invalid        | invalid        | invalid        |
+----------------------------+----------------+----------------+----------------+

When a certain byte string is valid in two or more encodings (and has the same
meaning when decoded), the encoder may use any applicable tag defined in this
document when encoding the string.

The decoder, assuming it supports all the variant tags (which is not required),
should arrive at the same decoded value regardless of which tag was chosen.
The tag used should not affect the decoded result value, i.e. the codepoint
sequence.  For example, these two tagged byte strings (tagged CESU-8 and
WTF-8, respectively) should have the same decoded value and be treated
identically::

  d9 01 10
    46 66 6f 6f ed a0 80

  d9 01 11
    46 66 6f 6f ed a0 80

Suppose A is sending CBOR data to B, and that both use CESU-8 internally for strings.
This is one valid approach for encoding in A:

* If an input string is valid UTF-8, encode as standard CBOR UTF-8 text string.

* Else, if an input is valid CESU-8, encode as a CBOR byte string with tag 272 (CESU-8).

* Else, encode as a plain CBOR byte string (should not happen if inputs are CESU-8).

However, this is also a valid approach:

* If an input string is valid UTF-8, encode as standard CBOR UTF-8 text string.

* Else, if an input is valid WTF-8, encode as a CBOR byte string with tag 273
  (WTF-8).

* Else, if an input is valid CESU-8 (but invalid WTF-8), encode as a CBOR byte
  string with tag 272 (CESU-8).

* Else, encode as a plain CBOR byte string (should not happen if inputs are CESU-8).

In both cases B would arrive at the same string result value.

References
==========

* [1] C. Bormann and P. Hoffman. "Concise Binary Object Representation (CBOR)".
  RFC 7049, October 2013.

* [2] R. McGowan (Ed.). "Unicode Technical Report #26: COMPATIBILITY ENCODING
  SCHEME FOR UTF-16: 8-BIT (CESU-8)".
  https://www.unicode.org/reports/tr26/tr26-4.html

* [3] S. Sapin. "The WTF-8 encoding".
  https://simonsapin.github.io/wtf-8/
  (archived: https://web.archive.org/web/20160524180037/https://simonsapin.github.io/wtf-8/)

* [4] Oracle. "Interface DataInput: Modified UTF-8".
  https://docs.oracle.com/javase/8/docs/api/java/io/DataInput.html#modified-utf-8

* [5] M. A. Lehmann. "Serialised language-independent object with type name and constructor arguments".
  http://cbor.schmorp.de/generic-object

Author
======

Sami Vaarala ``<sami.vaarala@iki.fi>``
