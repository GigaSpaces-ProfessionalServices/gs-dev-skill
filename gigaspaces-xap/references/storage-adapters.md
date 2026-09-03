# Property Storage Adapters

XAP keeps entries in memory, so a large property — an XML document, a JSON payload, a binary
blob — costs heap on every entry, whether or not it's ever queried. A **Property Storage
Adapter** lets a single property be stored in a transformed form (compressed, encrypted, or
otherwise re-encoded) while the rest of the application still reads and writes the original Java
type. The transform is transparent: XAP calls `toSpace()` on every write and `fromSpace()` on
every read, so no changes are needed in feeder or query code.

This is a different tool from indexing or `@SpaceDocument` — it doesn't make a blob queryable, it
makes it cheaper to hold. Keep any fields you need to filter or index on as separate plain
properties (see "Querying alongside an adapted property" below).

---

## Wiring it up

Annotate the getter with `@SpacePropertyStorageAdapter`, pointing at a `PropertyStorageAdapter`
implementation:

```java
import com.gigaspaces.annotation.pojo.*;
import java.io.Serializable;
import java.util.Map;

@SpaceClass
public class OrderDocument implements Serializable {

    private String id;
    private XmlProperty orderData;       // large payload — stored compressed
    private Map<String, Object> orderDataKeyProps; // extracted queryable subset
    private String customerId;

    @SpaceId(autoGenerate = false)
    @SpaceRouting
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    @SpacePropertyStorageAdapter(CompressedXmlPropertiesAdapter.class)
    public XmlProperty getOrderData() { return orderData; }
    public void setOrderData(XmlProperty orderData) { this.orderData = orderData; }

    @SpaceIndex
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public Map<String, Object> getOrderDataKeyProps() { return orderDataKeyProps; }
    public void setOrderDataKeyProps(Map<String, Object> orderDataKeyProps) { this.orderDataKeyProps = orderDataKeyProps; }
}
```

`orderData` is the plain Java type (`XmlProperty`) as far as feeder/business code is concerned —
the adapter class controls what's actually physically stored.

---

## Implementing a `PropertyStorageAdapter`

```java
import com.gigaspaces.client.storage_adapters.BinaryWrapper;
import com.gigaspaces.client.storage_adapters.PropertyStorageAdapter;
import com.gigaspaces.internal.io.CompressedMarshObject;
import java.io.IOException;
import java.util.zip.Deflater;
import java.util.zip.Inflater;

public class CompressedXmlPropertiesAdapter extends PropertyStorageAdapter {

    @Override
    public String getName() { return "CompressedXmlProperties"; }

    // The type actually persisted in the space for this property.
    @Override
    public Class<?> getStorageClass() { return CompressedMarshObject.class; }

    // Whether XAP can still do equality / ordered matching on the stored (transformed) value
    // itself — not on the original type's sub-fields. Most compression/encryption adapters
    // support neither meaningfully; declare honestly based on what the transform preserves.
    @Override
    public boolean supportsEqualsMatching() { return true; }

    @Override
    public boolean supportsOrderedMatching() { return true; }

    // Called on every write: original type in, storage type out.
    @Override
    public Object toSpace(Object value) throws IOException {
        if (value == null) return null;
        XmlProperty xmlProperty = (XmlProperty) value;
        byte[] compressed = compress(xmlProperty.getXmlContent());
        return toBinaryWrapper(compressed);
    }

    // Called on every read: storage type in, original type out.
    @Override
    public Object fromSpace(Object value) throws IOException, ClassNotFoundException {
        if (value == null) return null;
        byte[] compressed = unwrapBinary(value);
        XmlProperty xmlProperty = new XmlProperty();
        xmlProperty.setXmlContent(decompress(compressed));
        return xmlProperty;
    }

    @Override
    public BinaryWrapper toBinaryWrapper(byte[] data) {
        return new CompressedMarshObject(data);
    }

    // ... compress()/decompress() using java.util.zip.Deflater/Inflater ...
}
```

`getStorageClass()` and `toBinaryWrapper()` matter because that's what actually gets
serialized/replicated/persisted — `CompressedMarshObject` (from `com.gigaspaces.internal.io`)
wraps the compressed `byte[]` as the space-native storage form; a `String`-returning
`getStorageClass()` is the base64-encoded alternative when the storage tier can't hold raw
binary.

---

## Querying alongside an adapted property

The adapted property's stored form (compressed bytes) generally can't be queried on its internal
structure — only whole-value equality/ordering, per `supportsEqualsMatching()` /
`supportsOrderedMatching()`, and only if the adapter actually preserves those semantics on the
transformed bytes. If callers need to filter or index on data that lives inside the blob (a
customer ID embedded in an XML payload, a status field in a JSON document), **extract those
fields into separate plain, indexed properties on the same POJO** at write time, as
`OrderDocument.customerId` and `orderDataKeyProps` do above. The adapter only controls how the
*single annotated property* is physically stored — it doesn't replace normal indexing for
anything else on the class.

---

## When to Use / When Not to Use

| Situation | Recommendation |
|---|---|
| A property holds a large XML/JSON/binary payload that's rarely queried but consumes significant heap across many entries | ✅ Use a `PropertyStorageAdapter` to compress (or otherwise transform) it |
| Callers need to filter/index on data embedded inside that payload | ✅ Also extract the needed fields into separate plain properties — the adapter alone won't make them queryable |
| The property is small, or is queried on its own sub-fields frequently | ❌ Skip the adapter — use `@SpaceDocument`/nested indexed fields instead so the query engine can see the structure directly |
| You need field-level encryption at rest for compliance reasons | ✅ Same mechanism (`toSpace`/`fromSpace`), an encrypting adapter instead of a compressing one |

---

## Anti-Patterns

| ❌ Don't | ✅ Do instead |
|---|---|
| Rely on a compressed/encrypted property for `SQLQuery` filters on its internal fields | Extract queryable sub-fields into separate plain, indexed properties before/alongside adapting the blob |
| Declare `supportsEqualsMatching()`/`supportsOrderedMatching()` as `true` without verifying the transform actually preserves that semantics on the stored bytes | Return `false` unless equality/ordering on the *transformed* value is meaningful — a wrong `true` produces silently incorrect query results |
| Put transform state (e.g. compression level) in non-transient instance fields expecting it to travel per-value | Configure the adapter once via its own fields/constructor — `PropertyStorageAdapter` instances are shared, not per-entry |
