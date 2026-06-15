# BoroCTF - Geopro 5 Writeup

## Challenge Information

* **Challenge:** Geopro 5
* **Category:** OSINT / GeoGuessr
* **Points:** 100
* **Author:** ForeverFlames

### Description

> What country am I in?
>
> Note: ONLY 5 GUESSES
>
> Flag format: `boroCTF{country}`

A single image (`Country.png`) was provided.

---

# Initial Analysis

The challenge provides a Google Street View image of several automotive businesses located in a dry, mountainous environment.

At first glance, several clues immediately stand out:

* Arabic text appears on multiple signs.
* English translations are provided below the Arabic text.
* Businesses include:

  * Car Cleaning & Polishing
  * New Auto Spare Parts Sale
  * Repair Tires & Rims
* A phone number is visible:

```
95967008
```

---

# Investigating the Phone Number

The strongest clue is the phone number format.

The number:

```
95967008
```

contains 8 digits and begins with **95**.

This format is commonly used by mobile numbers in **Oman**.

Many neighboring Gulf countries use different numbering schemes:

| Country      | Typical Mobile Format             |
| ------------ | --------------------------------- |
| UAE          | 05x xxxxxxx                       |
| Saudi Arabia | 05x xxxxxxxx                      |
| Qatar        | 3xxx xxxx / 5xxx xxxx             |
| Kuwait       | 5xxx xxxx / 6xxx xxxx / 9xxx xxxx |
| Oman         | 9xxx xxxx                         |

The visible number strongly suggested Oman.

---

# Environmental Clues

Additional evidence supported this conclusion:

### Architecture

The buildings resemble small commercial garages commonly seen in Omani industrial zones.

### Language

Most signs use:

* Arabic
* English

which is standard throughout Oman.

### Terrain

The background contains rocky desert mountains.

This closely resembles the landscape surrounding many Omani cities such as:

* Muscat
* Nizwa
* Sohar
* Rustaq

where mountains and arid terrain are common.

### Roadside Construction Style

The paved forecourt, shop layout, and industrial-zone appearance are all highly characteristic of Oman.

---

# Conclusion

Combining:

* Omani phone number format (`95xxxxxx`)
* Arabic/English signage
* Mountainous desert environment
* Commercial architecture

the country could be confidently identified as:

**Oman**

---

# Flag

```text
boroCTF{oman}
```

---

# Lessons Learned

When solving GeoGuessr-style OSINT challenges:

1. Phone numbers are often the strongest clue.
2. Language alone may identify a region but not a specific country.
3. Terrain and architecture help eliminate neighboring countries.
4. Small business signs frequently contain local numbering formats that can uniquely identify a location.

In this challenge, the phone number was enough to narrow the search to Oman almost immediately.

