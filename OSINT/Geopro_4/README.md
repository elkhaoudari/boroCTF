# BoroCTF 2025 - Geopro 4 Writeup

## Challenge Information

* **Challenge:** Geopro 4
* **Category:** OSINT / GeoGuessr
* **Points:** 100
* **Author:** ForeverFlames

### Description

> Find the streetname. :D
>
> Flag format: `boroCTF{street_name}`

A single Street View image was provided, and the goal was to identify the street name where the image was taken.

---

# Initial Analysis

The image showed several useful clues:

* A building labeled **MEDICUS Diagnostic Center**
* A nearby establishment called **Pizza Junction**
* Tricycles and road infrastructure typical of the **Philippines**

Since both businesses appeared clearly in the image, they became the primary OSINT targets.

---

# Identifying the Location

The first step was to search for:

```
Medicus Diagnostic Center Pizza Junction
```

Several Medicus branches exist throughout the Philippines, so the challenge was determining which one matched the screenshot.

After searching for combinations of:

```
"Medicus Diagnostic Center" "Pizza Junction"
"Pizza Junction near Medicus"
"Medicus Roxas"
```

a strong lead appeared:

**Pizza Junction, Roxas City, Capiz**

Multiple references indicated that Pizza Junction is located near Medicus Diagnostic Center in Roxas City.

Further investigation revealed:

```
Medicus Diagnostic Center
Rizal Street
Roxas City, Capiz
Philippines
```

The surrounding buildings and storefront arrangement matched the challenge image.

---

# Verification

Cross-referencing maps, business listings, and photos confirmed that:

* Medicus Diagnostic Center sits on **Rizal Street**
* Pizza Junction is directly nearby
* Building positions matched the provided Street View image

Therefore the street name was:

```
Rizal
```

---

# Flag

```text
boroCTF{Rizal}
```

---

# Lessons Learned

* Business signs are often enough to solve GeoGuessr-style challenges.
* Combining multiple visible landmarks dramatically narrows the search space.
* Local business directories and map services can be extremely useful for OSINT.
* When a challenge asks for a street name, verifying the exact branch location is critical because many businesses have multiple locations.

---

# Tools Used

* Google Search
* Google Maps
* Business directories
* Street View comparison
* Manual OSINT techniques

---

# Conclusion

This was a straightforward OSINT challenge. The visible signs for **Medicus Diagnostic Center** and **Pizza Junction** provided enough information to locate the exact area in **Roxas City, Capiz, Philippines**. Once the branch was identified, the street name **Rizal Street** was easily obtained, yielding the flag.

Flag: `boroCTF{Rizal}`

