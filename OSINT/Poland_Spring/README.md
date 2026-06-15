# Poland Spring

**Category:** OSINT
**Difficulty:** Easy

## Description

> who even makes these?
>
> Flag Format:
>
> `boroCTF{tag_maker}`

The challenge provided a single image containing a product tag.

---

## Initial Analysis

At first glance, the image appeared to show a damaged or detached product label. Since the challenge was categorized as OSINT, the most promising lead was the barcode printed on the tag.

After zooming in, the UPC number could be clearly identified as:

```text
075720481279
```

---

## Investigation

The UPC was searched using publicly available product lookup services and search engines.

The search results consistently identified the product as:

```text
Poland Spring Natural Spring Water
```

At this point, the challenge description became much clearer.

The prompt asked:

> who even makes these?

Rather than asking for a parent company or manufacturer, the intended answer was the product brand associated with the barcode.

---

## Solution

1. Inspect the image.
2. Extract the UPC barcode number.
3. Search the UPC online.
4. Identify the associated product brand.
5. Submit the brand name using the required flag format.

---

## Flag

```text
boroCTF{poland_spring}
```

---

## Lessons Learned

* Barcodes and UPC numbers are valuable OSINT indicators.
* Public product databases can quickly identify commercial products.
* Many beginner OSINT challenges rely on extracting information from seemingly insignificant visual details.

---

## Tools Used

* Image Viewer
* Google Search
* UPC Lookup Databases
* Browser Zoom

## References

* UPC Product Lookup
* GS1 Barcode Standards

