# IPIP-328: JSON and CBOR Response Formats on HTTP Gateways

- Start Date: 2022-10-07
- Related Issues:
  - [ipfs/in-web-browsers/issues/182]
  - [ipfs/specs/pull/328]
  - [ipfs/kubo/issues/8823]
  - [ipfs/kubo/pull/9335]
  - [ipfs/go-ipfs/issues/7552]

## Summary

Add support for the [DAG-JSON], [DAG-CBOR], JSON and CBOR response formats in
the [HTTP Gateway](../http-gateways/).

## Motivation

Currently, the gateway supports requesting data in the [DAG-PB], RAW, [CAR] and
TAR formats. In addition, it allows for traversing of links encoded through CBOR
Tag 42, as long as they are intermediate links, and not the final document.
It works on both DAG-CBOR, and its JSON representation, DAG-JSON. However, it
should be possible to download deserialized versions of the final JSON/CBOR document
in raw format (not wrapped in UnixFS).

The main functional gap in the IPFS ecosystem is the lack of support for
non-UnixFS DAGs on HTTP gateways. Users are able to create custom DAGs based on
traversable DAG-CBOR thanks to [CBOR tag 42 being reserved for CIDs][cbor-42]
and DAG-JSON documents, but they are unable to load deserialized documents from
a local gateway, which is severely decreasing the utility of non-UnixFS DAGs.

Adding JSON and CBOR response types will also benefit UnixFS. DAG-PB has a
[logical format][dag-pb-format] which makes it possible to represent a DAG-PB
directory as a [DAG-JSON] document. This means that, if we support DAG-JSON in
the gateway, then we would support
[JSON responses for directory listings][ipfs/go-ipfs/issues/7552], which has been
requested by our users in the past.

In addition, this functionality is already present on the current Kubo CLI. By
bringing it to the gateways, we provide users with more power when it comes
to storing and fetching CBOR and JSON in IPFS.

## Detailed design

The solution is to allow the Gateway to support serializing data as [DAG-JSON],
[DAG-CBOR], JSON and CBOR by requesting them using either the `Accept` HTTP header
or the `format` URL query. In addition, if the resolved CID is of one of the
aforementioned types, the gateway should be able to resolve them instead of
failing with `node type unknown`.

## Test fixtures

- [`bafybeiegxwlgmoh2cny7qlolykdf7aq7g6dlommarldrbm7c4hbckhfcke`][f-dag-pb] is a
  DAG-PB directory.
- [`bafkreidmwhhm6myajxlpu7kofe3aqwf4ezxxn46cp5fko7mb6x74g4k5nm`][f-dag-pb-json]
  is the aforementioned DAG-PB directory's [Logical DAG-JSON representation][dag-pb-format] that
  is expected to be returned when using `?format=dag-json`.

## Design rationale

The current gateway already supports different response formats via the
`Accept` HTTP header and the `format` URL query. This IPIP proposes adding
JSON and CBOR formats to that list.

In addition, the current gateway already supports traversing through DAG-CBOR
and DAG-JSON links if they are intermediary documents. With this IPIP, we aim
to be able to download the DAG-CBOR, DAG-JSON, JSON and CBOR documents
themselves, with correct `Content-Type` headers.

### User benefit

The user benefits from this change as they will now be able to retrieve
content encoded in the traversable DAG-JSON and DAG-CBOR formats. This is
something that has been [requested before][ipfs/go-ipfs/issues/7552].

In addition, both UX and DX are significantly improved, since every UnixFS directory can
now be inspected in a regular web browser via `?format=json`. This can remove the
need for parsing HTML with directory listing.

### Compatibility

This IPIP adds new response types and does not modify existing ones,
making it a backwards-compatible change.

### Security

Serializers and deserializers for the JSON and CBOR must follow the security
considerations of the original specifications, found in:

- [RFC 8259 (JSON), Section 12][rfc8259-sec12]
- [RFC 8949 (CBOR), Section 10][rfc8949-sec10]

DAG-JSON and DAG-CBOR follow the same security considerations as JSON and CBOR.
Note that DAG-JSON and DAG-CBOR are stricter subsets of JSON and CBOR, respectively.
Therefore they must follow their specification and error if the payload is not
strict enough:

- [DAG-JSON Spec][dag-json-spec]
- [DAG-CBOR Spec][dag-cbor-spec]

### Alternatives

#### Why four content types?

If we do not introduce  DAG-JSON, DAG-CBOR, JSON and CBOR  response formats in
the gateway, the usage of IPFS is constricted to files and directories represented
by UnixFS (DAG-PB) codec. Therefore, if a user wants to store JSON and/or CBOR
in IPFS, they have to wrap it as a UnixFS file in order to be able to fetch it
through the gateway. That adds size and processing overhead.

In addition, we could introduce only DAG-JSON and DAG-CBOR. However, not
supporting the generic variants, JSON and CBOR, would lead to poor UX. The
ability to retrieve DAG-JSON as `application/json` is an important step
for the interoperability of the HTTP Gateway with web browsers and other tools
that expect specific Content Types. Namely, `Content-Type: application/json` with
`Content-Disposition: inline` allows for JSON preview to be rendered in a web browser
and webdev tools.

#### Why JSON/CBOR pathing is limited to full blocks?

Finally, we considered supporting pathing within both DAG and non-DAG variants
of the JSON and CBOR codecs. Pathing within these documents could lead to responses
with extracts from the document. For example, if we have the document:

```json
{
  "link" {
    "to": {
      "some": {
        "cid2": <cbor tag 42 pointing at different CID>
       }
    }
  }
}
```

With CID `bafy`, and we navigate to `/ipfs/bafy/link/to`, we would be able to
retrieve an extract from the document.

```json
{
  "some": {
    "cid2": <cbor tag 42 pointing at different CID>
    }
}
```

However, supporting this raises questions whose answers are not clearly defined
or agreed upon yet. Right now, pathing is only supported over CID-based Links,
such as Tag 42 in CBOR. In addition, some HTTP headers regarding caching are based
on the CID, and adding extraction pathings would not be clear. Giving users the
possibility to retrieve JSON, CBOR, DAG-JSON AND DAG-CBOR documents through the
gateway is, in itself, a progress and will open the doors for new tools and explorations.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[cbor-42]: https://github.com/core-wg/yang-cbor/issues/13#issuecomment-524378859
[DAG-PB]: https://ipld.io/docs/codecs/known/dag-pb/
[dag-pb-format]: https://ipld.io/specs/codecs/dag-pb/spec/#logical-format
[DAG-JSON]: https://ipld.io/docs/codecs/known/dag-json/
[DAG-CBOR]: https://ipld.io/docs/codecs/known/dag-cbor/
[CAR]: https://ipld.io/specs/transport/car/
[ipfs/in-web-browsers/issues/182]: https://github.com/ipfs/in-web-browsers/issues/182
[ipfs/specs/pull/328]: https://github.com/ipfs/specs/pull/328
[ipfs/kubo/issues/8823]: https://github.com/ipfs/kubo/issues/8823
[ipfs/kubo/pull/9335]: https://github.com/ipfs/kubo/pull/9335
[ipfs/go-ipfs/issues/7552]: https://github.com/ipfs/go-ipfs/issues/7552
[f-dag-pb]: https://dweb.link/ipfs/bafybeiegxwlgmoh2cny7qlolykdf7aq7g6dlommarldrbm7c4hbckhfcke
[f-dag-pb-json]: https://dweb.link/ipfs/bafkreidmwhhm6myajxlpu7kofe3aqwf4ezxxn46cp5fko7mb6x74g4k5nm
[rfc8259-sec12]: https://datatracker.ietf.org/doc/html/rfc8259#section-12
[rfc8949-sec10]: https://datatracker.ietf.org/doc/html/rfc8949#section-10
[dag-json-spec]: https://ipld.io/specs/codecs/dag-json/spec/
[dag-cbor-spec]: https://ipld.io/specs/codecs/dag-cbor/spec/
