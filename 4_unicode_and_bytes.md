# Unicode and Bytes

## Code Points vs Byte Representations

```python
# Identity (code point) is separate from byte representation
char = 'A'
print(ord(char))          # 65 — code point (U+0041)
print(hex(ord(char)))     # 0x41

# Same code point, different byte sequences depending on encoding
print(char.encode('utf-8'))       # b'\x41'
print(char.encode('utf-16-le'))   # b'\x41\x00'
```

```python
# Code points range from U+0000 to U+10FFFF
print(ord('€'))   # 8364 (U+20AC)
print(ord('𩸽'))  # 171581 (U+29E3D) — beyond BMP, uses surrogate pair in some encodings
```

---

## Handling Encoding Errors

```python
text = "café"

# Default: strict — raises UnicodeDecodeError on invalid bytes
broken = b'caf\xe9'  # \xe9 is not valid UTF-8
try:
    broken.decode('utf-8')
except UnicodeDecodeError as e:
    print(f"Error: {e}")

# xmlcharrefreplace — preserves data when you can't afford to lose it
result = broken.decode('utf-8', errors='xmlcharrefreplace')
print(result)  # caf&#233;  — numeric XML character reference
```

```python
# Other error handlers
print(broken.decode('utf-8', errors='replace'))   # caf
print(broken.decode('utf-8', errors='ignore'))    # caf
```

---

## UTF-8 Design: Accidental Decoding is Unlikely

```python
# Arbitrary bytes > 127 almost never decode as valid UTF-8 by accident
import random

random_bytes = bytes(random.randint(128, 255) for _ in range(10))
try:
    text = random_bytes.decode('utf-8')
    print("Decoded successfully — likely genuine UTF-8")
except UnicodeDecodeError:
    print("Failed to decode — not valid UTF-8")
```

---

## The Unicode Sandwich (Best Practice)

```python
# ✅ Correct: decode early, process as str, encode late
def process_file(input_path, output_path):
    # Decode bytes → str as early as possible
    with open(input_path, 'r', encoding='utf-8') as f:
        content = f.read()  # str
    
    # All processing works with str
    processed = content.upper()
    
    # Encode str → bytes as late as possible
    with open(output_path, 'w', encoding='utf-8') as f:
        f.write(processed)

# ❌ Avoid: mixing bytes and str in business logic
```

---

## Always Pass Encoding Explicitly

```python
# ❌ Relying on system default — different across platforms
with open('data.txt', 'r') as f:
    content = f.read()  # encoding depends on locale

# ✅ Explicit encoding
with open('data.txt', 'r', encoding='utf-8') as f:
    content = f.read()  # predictable behaviour
```

---

## Unicode Normalisation

### NFC vs NFD

```python
# é can be represented two ways
composed = '\u00e9'        # single code point: é
decomposed = '\u0065\u0301' # e + combining acute accent: é

print(composed, len(composed))      # é 1
print(decomposed, len(decomposed))  # é 2
print(composed == decomposed)       # False — different code point sequences!
```

```python
import unicodedata

# NFC: composes to shortest form
nfc_composed = unicodedata.normalize('NFC', decomposed)
print(len(nfc_composed))  # 1

# NFD: decomposes to base + combining characters
nfd_decomposed = unicodedata.normalize('NFD', composed)
print(len(nfd_decomposed))  # 2

# W3C recommends NFC for storage
```

### Keyboard Drivers Produce Composed Characters

```python
# Typing 'é' on a keyboard typically produces NFC
typed = 'é'
print(len(typed))                              # 1
print(unicodedata.normalize('NFD', typed))     # e + combining accent
```

---

## Compatibility Characters

```python
# U+00B5 MICRO SIGN (added for latin1 compatibility)
# U+03BC GREEK SMALL LETTER MU (canonical form)
micro = '\u00b5'
mu = '\u03bc'
print(micro == mu)  # False — different code points, same visual appearance
```

### NFKC / NFKD: Replace Compatibility Characters

```python
# NFKC/NFKD normalise compatibility characters to preferred form
print(unicodedata.normalize('NFKC', micro))  # μ (U+03BC)
print(unicodedata.normalize('NFKD', micro))  # μ (U+03BC)

# ⚠️ Data loss: original code point is lost
# Use only for search/indexing, not storage
```

```python
# Another example: ligatures
ligature = '\ufb00'  # ﬀ (LATIN SMALL LIGATURE FF)
print(unicodedata.normalize('NFKC', ligature))  # ff (two characters)
```

---

## Case Folding

```python
# str.casefold() is more aggressive than str.lower()
# ~300 code points where they differ

# German sharp s
print('ß'.lower())     # ß
print('ß'.casefold())  # ss

# Greek final sigma
print('Σ'.lower())     # σ
print('ς'.casefold())  # σ
```

```python
# Case-insensitive comparison: normalise NFC first, then casefold
def case_insensitive_equal(a, b):
    nfc_a = unicodedata.normalize('NFC', a)
    nfc_b = unicodedata.normalize('NFC', b)
    return nfc_a.casefold() == nfc_b.casefold()

print(case_insensitive_equal('Straße', 'STRASSE'))  # True
```

---

## Removing Diacritics

```python
import unicodedata

def remove_diacritics(text):
    # 1. NFD normalise (decompose into base + combining chars)
    nfd = unicodedata.normalize('NFD', text)
    # 2. Filter out combining characters (category starts with 'M')
    filtered = ''.join(c for c in nfd if not unicodedata.category(c).startswith('M'))
    # 3. NFC normalise back
    return unicodedata.normalize('NFC', filtered)

print(remove_diacritics('café résumé naïve'))  # cafe resume naive
```

---

## Character Translation

```python
# str.maketrans + str.translate for custom character mapping
translation = str.maketrans({'a': '4', 'e': '3', 'o': '0'})
print('hello world'.translate(translation))  # h3ll0 w0rld

# Remove specific characters by mapping to None
remove_vowels = str.maketrans('', '', 'aeiou')
print('hello world'.translate(remove_vowels))  # hll wrld
```

---

## Locale-Sensitive Sorting

```python
# Standard Python sorting uses code point order
words = ['café', 'cafe', 'côte', 'cote']
print(sorted(words))  # ['cafe', 'café', 'cote', 'côte'] — not locale-aware

# For locale-sensitive sorting, use pyuca or pyICU
# pip install pyuca
import pyuca
collator = pyuca.Collator()
print(sorted(words, key=collator.sort_key))
# ['cafe', 'café', 'cote', 'côte'] — accents handled properly
```

```python
# pyICU supports locale-specific collation tables
# pip install pyicu
# import icu
# collator = icu.Collator.createInstance(icu.Locale('fr_FR'))
```

---

## Unicode Database

```python
import unicodedata

# Character metadata from the Unicode database
print(unicodedata.name('€'))       # EURO SIGN
print(unicodedata.category('€'))   # Sc (Symbol, Currency)
print(unicodedata.decimal('5'))    # 5
print(unicodedata.numeric('Ⅷ'))    # 8.0

# Used by str methods internally
print('5'.isnumeric())   # True
print('Ⅷ'.isnumeric())   # True
print('€'.isnumeric())   # False
print('a'.isprintable()) # True
```

---

## Regular Expressions: str vs bytes

```python
import re

# str regex: recognises non-ASCII word and digit characters
print(re.findall(r'\w+', 'café 123 résumé'))  # ['café', '123', 'résumé']
print(re.findall(r'\d+', '123 ٤٥٦'))         # ['123', '٤٥٦']  — Arabic digits

# bytes regex: only ASCII word and digit characters
print(re.findall(rb'\w+', b'caf\xc3\xa9 123'))  # [b'caf', b'123']
# \xc3\xa9 is UTF-8 for é, but not matched by \w in bytes mode
```

---

## `os` Module: str vs bytes Filenames

```python
import os, sys

# str paths: automatically encoded/decoded using filesystem encoding
print(sys.getfilesystemencoding())  # e.g., 'utf-8'

# str in → str out
files = os.listdir('.')
print(files)  # list of str

# bytes in → bytes out (you handle encoding)
files_bytes = os.listdir(b'.')
print(files_bytes)  # list of bytes — your responsibility to decode
```

---

## Encoding Detection with `chardet`

```python
# When metadata is absent, chardet can guess the encoding
# pip install chardet
import chardet

unknown = b'\xc3\xa9l\xc3\xa8ve'  # UTF-8 for "élève"
result = chardet.detect(unknown)
print(result)  # {'encoding': 'utf-8', 'confidence': 0.99, ...}

decoded = unknown.decode(result['encoding'])
print(decoded)  # élève
```
