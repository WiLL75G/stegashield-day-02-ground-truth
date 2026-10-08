
# Dataset and Ground Truth Preparation

## Overview

Day 2 establishes the dataset and ground truth foundation for my StegaShield detection validation pilot before any controlled image is submitted to the detection engine.

The goal was to understand the reference dataset, identify possible validation limitations, independently create known clean and LSB-modified images, and verify that the resulting artifacts could be reproduced.

This gives me a controlled starting point for evaluating StegaShield without allowing its predictions to determine the correct labels.

---

## Dataset Strategy

The pilot separates two sources of image data.

**Dataset A — Reference Dataset**

The StegaShield team identified the following dataset as material used during their training and testing:

https://www.kaggle.com/datasets/marcozuppelli/stegoimagesdataset/data

This dataset provides reference material for understanding the model's intended detection scope.

However, because it was used during model development, results from that dataset alone cannot establish independent generalization.

**Dataset B — Independent Dataset**

Dataset B is created separately for this pilot.

Its purpose is to provide samples with known ground truth established before StegaShield analyzes them.

The first controlled pair contains:

- One clean RGB image
- One modified image derived from that same clean parent
- One controlled raw payload
- One RGB least significant bit embedding method

The independent dataset currently contains one verified pair.

This is sufficient to establish the initial testing procedure, but not sufficient to measure overall detection accuracy.

---

# Lab Role During Day 2

The Day 1 baseline established three systems:

**Mac M2 Host**
- SOC analysis system
- Splunk Enterprise
- Lab management

**Windows Endpoint**
- Controlled endpoint
- Sysmon
- Windows Event Logs
- Splunk Universal Forwarder

**Ubuntu Server**
- Controlled server
- Dataset generation
- Ground truth preparation
- Future StegaShield host
- Future HTTPS receiver
- Future Zeek sensor

Day 2 work was performed on Ubuntu.

The purpose was not to test network transfers or correlate SOC telemetry yet.

The purpose was to establish trustworthy image samples before introducing those additional components.

---

# Independent Dataset Workspace

The Dataset B workspace was created under:

```text
~/stegashield-pilot/dataset-b-independent/
```

The investigation used the following directory structure:

```text
dataset-b-independent/
├── clean/
├── modified/
├── scripts/
└── evidence/
```

Each directory has a specific purpose.

**clean/**

Stores original images that have not intentionally undergone the controlled embedding process.

**modified/**

Stores images generated from known clean parents using a documented modification method.

**scripts/**

Stores the Python programs used to generate and modify the images.

**evidence/**

Stores the manifest, SHA-256 records, reproduced artifacts, and verification evidence.

### Why this matters

The clean parent and modified image must remain distinguishable.

If the original image is overwritten, the investigation loses its controlled comparison.

Separating the files also makes it easier to verify that a later detector result refers to the intended sample.

---

# Clean Image Generation

The first independent clean image was generated using Python and Pillow.

The generator used:

```text
Image dimensions: 512 × 512
Color mode: RGB
Image format: PNG
Random seed: 20261007
```

The script was saved as:

```text
scripts/generate_clean.py
```

The generated image was saved as:

```text
clean/SS-001-CLEAN.png
```

The generator creates deterministic pseudorandom RGB pixel values.

Using a fixed seed means that running the same script with the same parameters should produce the same pixel data.

The script also refuses to overwrite an existing output file.

![Clean image generation and reproduction](evidence/01-clean-image-generation.png)

### Why this matters

The clean image was created independently of StegaShield.

Its ground truth was established from the controlled generation procedure, not from a model prediction.

Using RGB for the independent clean parent also ensures that its modified counterpart can retain the same image mode.

### Limitation

This image contains synthetic pseudorandom pixel values.

It is not representative of an ordinary photograph.

That distinction matters because image statistics can influence steganalysis models.

The results from this first sample cannot automatically be generalized to natural images.

---

# Controlled LSB Embedding

The next step was to create a modified image from the clean parent.

The embedding script was saved as:

```text
scripts/embed_lsb.py
```

The source image was:

```text
clean/SS-001-CLEAN.png
```

The modified output was:

```text
modified/SS-001-LSB-RAW.png
```

The controlled payload was:

```text
STEGASHIELD_PILOT_SS001
```

The payload was 23 bytes.

The script first encodes the payload length using a 32-bit header.

It then converts the payload into bits.

The total number of positions used for embedding was:

```text
32-bit length header
+
184 payload bits
=
216 embedded bit positions
```

The script modifies the least significant bits of successive RGB channel values.

The central operation is:

```python
channels[index] = (channels[index] & 254) | int(bit)
```

The operation clears the lowest bit and writes the intended payload bit.

The remaining seven bits are preserved by this operation.

![Controlled LSB embedding](evidence/02-lsb-embedding.png)

### Why this matters

The modified image has a known clean parent.

The payload is known.

The payload preparation method is known.

The embedding method is known.

This establishes the basis for comparing future StegaShield results against independently assigned ground truth.

### Important distinction

Raw, Base64, and ZIP describe different ways of preparing payload data.

LSB describes how the data is embedded into image pixels.

Day 2's verified independent sample uses raw payload preparation and RGB LSB embedding.

Base64 and ZIP variants are not part of this verified Dataset B pair.

---

# Ground Truth Labels

The two test samples were assigned the following labels:

| Test ID | Ground Truth | Parent |
|---|---|---|
| `SS-001-CLEAN` | CLEAN | None |
| `SS-001-LSB-RAW` | MODIFIED | `SS-001-CLEAN` |

The first image is the original controlled carrier.

The second image was generated from that carrier by the LSB embedding script.

The modified label does not mean that the image is malicious.

It means that the image underwent the documented embedding operation.

### Why this matters

StegaShield will later produce its own predictions.

Those predictions must be compared against the established labels.

A model score must not become the source of ground truth.

Otherwise, the investigation would be using the detector to validate itself.

---

# LSB Verification Scope

The embedding script documents the controlled transformation.

It uses the clean parent as input, writes the length header and payload bits, and saves a separate modified PNG.

The operation targets only the least significant bit of each selected RGB channel.

However, the number of embedded bit positions is not necessarily the number of channel values that changed.

For example, writing a `1` into a channel whose lowest bit is already `1` does not change that channel value.

The verified embedding procedure used 216 bit positions.

This does not mean 216 channels necessarily changed.

![LSB verification evidence](evidence/03-lsb-pixel-verification.png)

### What this proves

The preserved script establishes how the controlled modification was performed.

The generated output and reproducibility checks establish that the process produced a stable modified artifact.

### What remains separate

A full extraction test or an independently documented pixel-difference count should be reported only when supported by its actual verification output.

The embedding script alone does not establish a measured changed-pixel count or successful extraction by a separate decoder.

---

# SHA-256 Integrity Baseline

After generating the images, SHA-256 fingerprints were recorded.

The clean image produced:

```text
Test ID:
SS-001-CLEAN

SHA-256:
f8d20edc5ca3b65273adc0bd201391fca3293f60e6c29f56730fb2310c925e61
```

The modified image produced:

```text
Test ID:
SS-001-LSB-RAW

SHA-256:
98c5d60f0e3d1b9540ece2586c5f6677ab08a48a0461686ccac50a885c35f0f6
```

The recorded hash files were:

```text
evidence/SS-001-CLEAN.sha256
evidence/SS-001-LSB-RAW.sha256
```

### Why this matters

A SHA-256 fingerprint identifies the exact bytes of a file.

If the file changes, its digest will ordinarily change.

This allows the investigation to detect accidental modifications to the test artifacts.

However:

```text
Different hashes ≠ proof of steganography
```

The hashes establish file identity and integrity.

The controlled embedding procedure establishes why the modified image was created.

---

# Clean Image Reproducibility

The clean image generator was executed again using a separate output path.

The reproduced file was saved as:

```text
evidence/SS-001-REPRODUCED.png
```

The reproduced clean image generated the same SHA-256 digest as the original clean image.

This confirmed byte-for-byte reproduction of the clean artifact under the tested conditions.

### Why this matters

The result demonstrates that the saved generator and its fixed seed can recreate the same clean sample.

The ground truth does not depend on an image that can no longer be reproduced.

---

# Modified Image Reproducibility

The LSB embedding script was executed again against the original clean parent.

The reproduced modified image was saved as:

```text
evidence/SS-001-LSB-REPRODUCED.png
```

The original modified image and reproduced modified image generated identical SHA-256 fingerprints:

```text
98c5d60f0e3d1b9540ece2586c5f6677ab08a48a0461686ccac50a885c35f0f6
```

The checksum verification returned `OK` for the original and reproduced modified artifacts.

The reproducibility checksum record was saved as:

```text
evidence/SS-001-reproducibility.sha256
```

![SHA-256 reproducibility verification](evidence/04-sha256-reproducibility.png)

## What this proves

The saved embedding script reproduced the same modified PNG from the same clean parent.

This supports the repeatability of the controlled test condition.

It does not prove that StegaShield can detect the embedded payload.

That remains a separate model-evaluation question.

---

# Evidence Protection

Both image-generation scripts contain overwrite protection.

When an existing output path was supplied, the scripts raised a `FileExistsError`.

This behavior was intentional.

It prevents a later run from silently replacing an established test artifact.

Subsequent checksum verification confirmed that the recorded artifacts remained consistent.

### Why this matters

In an investigation, preserving the original test material is important.

If a file is replaced without documentation, its earlier ground-truth record may no longer refer to the same bytes.

Overwrite protection reduces that risk.

---

# Ground Truth Manifest

The manifest was saved as:

```text
evidence/ground-truth-manifest.csv
```

Its fields are:

```text
test_id
parent_id
relative_path
ground_truth
embedding_method
payload_preparation
payload_bytes
sha256
```

The manifest contains two records.

| Test ID | Relative Path | Ground Truth | Embedding Method | Payload Bytes |
|---|---|---|---|---:|
| `SS-001-CLEAN` | `clean/SS-001-CLEAN.png` | CLEAN | none | 0 |
| `SS-001-LSB-RAW` | `modified/SS-001-LSB-RAW.png` | MODIFIED | LSB_RGB | 23 |

The modified sample references the clean sample through its parent ID.

This preserves the relationship between the original carrier and the controlled modification.

---

# Manifest Integrity Verification

The manifest was validated using Python.

The verification process:

1. Read the CSV records.
2. Resolved each relative path against the Dataset B root.
3. Checked that each referenced file existed.
4. Recalculated its SHA-256 digest.
5. Compared the calculated digest against the manifest value.

The output was:

```text
SS-001-CLEAN PASS
SS-001-LSB-RAW PASS
Records checked: 2
```

![Ground truth manifest validation](evidence/05-ground-truth-validation.png)

## What this proves

Both manifest entries referenced existing files.

Both files matched their recorded SHA-256 fingerprints.

The manifest and current artifacts were consistent at the time of verification.

This does not establish statistical model performance.

It establishes the integrity of the initial independent test pair.

---

# Engineering Decisions

Several technical decisions were made during Day 2.

## 1. Separate Independent Ground Truth From Reference Data

**What**

Create Dataset B independently rather than relying only on the team-linked dataset.

**Why**

The team identified the reference dataset as material used during training and testing.

Using only that dataset could weaken claims of independent generalization.

**Alternative**

Use the existing reference dataset for every test.

**Evidence**

The team's stated training/testing history establishes the overlap concern.

**What would change**

Without independently generated samples, later model results could be difficult to interpret as genuinely independent validation.

---

## 2. Preserve the Clean Parent

**What**

Generate the modified image as a separate file.

**Why**

The investigation needs a stable original for comparison.

**Alternative**

Modify the original file in place.

**Evidence**

The scripts generated distinct clean and modified paths.

**What would change**

Overwriting the parent would remove the controlled reference image and weaken reproducibility.

---

## 3. Use Deterministic Generation

**What**

Use a fixed random seed for the synthetic clean carrier.

**Why**

The same image must be reproducible.

**Alternative**

Generate a new random image on every execution.

**Evidence**

The reproduced clean image matched the original SHA-256 digest.

**What would change**

Without deterministic generation, repeating the script would not reliably reproduce the same artifact.

---

## 4. Isolate One Embedding Condition

**What**

Start with a raw 23-byte payload embedded through RGB LSB substitution.

**Why**

The first experiment should have a controlled and understandable modification.

**Alternative**

Introduce raw, Base64, ZIP, and other methods simultaneously.

**Evidence**

The saved embedding script documents the exact operation and payload.

**What would change**

Changing several variables together would make future detector behavior harder to attribute to a particular condition.

---

## 5. Preserve Integrity Evidence

**What**

Record SHA-256 fingerprints and protect output paths from overwriting.

**Why**

The test files must remain stable and identifiable.

**Alternative**

Depend only on filenames and manually regenerated artifacts.

**Evidence**

Checksum verification passed and the scripts rejected existing output paths.

**What would change**

Undocumented file replacement could invalidate later comparisons.

---

# Day 2 Analysis

## Observed

- An independent Dataset B workspace was established on Ubuntu.
- A synthetic 512 × 512 RGB PNG was generated using a fixed random seed.
- The clean image was saved as `SS-001-CLEAN.png`.
- A 23-byte controlled payload was embedded into a separate image using RGB LSB substitution.
- The embedding procedure used 216 bit positions, including the payload-length header.
- The modified image was saved as `SS-001-LSB-RAW.png`.
- Both original images received SHA-256 fingerprints.
- The clean image was reproduced with an identical digest.
- The modified image was reproduced with an identical digest.
- Existing-file overwrite attempts were rejected.
- Both ground-truth manifest records passed integrity verification.

---

## Correlated

The preserved generation scripts, clean parent, modified image, and SHA-256 records support the following evidence chain:

```text
Deterministic Clean Image Generator
              |
              v
       SS-001-CLEAN
              |
              v
     Controlled LSB Embedding
              |
              v
       SS-001-LSB-RAW
              |
              v
      SHA-256 Verification
              |
              v
     Ground Truth Manifest
```

The reproduction checks establish that the original outputs can be regenerated under the tested conditions.

The manifest checks establish that the current files match their recorded fingerprints.

These observations support the integrity of the independent sample pair.

---

## Interpretation

Day 2 established a controlled foundation for evaluating StegaShield.

The clean and modified labels were assigned independently of the detection engine.

The image-generation procedure was preserved.

The files were fingerprinted.

The original outputs were reproduced.

The manifest was checked against the actual files.

Most importantly:

```text
Ground truth ≠ model prediction

File integrity ≠ steganography detection

LSB embedding ≠ malicious intent

Modified image ≠ confirmed exfiltration
```

Each claim requires its own supporting evidence.

---

# Unknown

Day 2 does not answer whether:

- StegaShield will classify the clean image correctly.
- StegaShield will classify the modified image correctly.
- The model will produce repeatable probability scores.
- The synthetic carrier will influence the model's prediction.
- Different payload sizes will change detection results.
- Base64 or ZIP payload preparation will affect model behavior.
- Natural photographs will produce comparable results.
- The model will produce false positives or false negatives.
- Network telemetry will provide useful context for an image investigation.
- StegaShield results can be correlated effectively with Splunk, Sysmon, and Zeek.

Those questions belong to later stages of the pilot.

---

# Evidence Gaps

Several limitations remain.

### Dataset Size

Dataset B currently contains one independently verified clean/modified pair.

This is not enough to calculate meaningful overall detection-performance metrics.

### Carrier Diversity

The first carrier is a synthetic pseudorandom RGB image.

Natural images and other carrier types have not yet been established as independent test populations.

### Payload Variants

The verified sample uses raw payload preparation.

Base64 and ZIP variants have not yet been independently generated and verified in Dataset B.

### Extraction and Pixel-Level Measurements

The documented embedding script establishes the intended operation.

A separately evidenced extraction result and measured changed-pixel count should be included only if the corresponding verification outputs are available.

### Detector Results

No StegaShield probability scores or classifications have been collected during Day 2.

No detection-performance conclusion can be made yet.

---

# Day 2 Disposition

**Proceed to Day 3: StegaShield Installation and Clean Baseline Testing.**

The first independent clean/modified pair has been generated and verified.

The original and reproduced files match their recorded SHA-256 fingerprints.

The ground-truth manifest passed verification for both records.

The technical preparation for the first controlled sample pair is complete.

The public Day 2 repository should be marked complete only after the README, screenshots, scripts, manifest, and published evidence paths have been verified.

---

# Day 2 Lesson

Independent ground truth must exist before evaluating a detection engine.

During this stage I established:

- a controlled clean image
- a known LSB-modified counterpart
- a documented payload and embedding method
- a preserved clean-parent relationship
- SHA-256 fingerprints
- reproducible generation scripts
- a validated ground-truth manifest

The important lesson is that a detector cannot be meaningfully evaluated by treating its own predictions as the correct answers.

The correct labels must come from independent evidence.

That is what Day 2 establishes for the first controlled sample pair.

---

# Evidence

The Day 2 public screenshot evidence is organized as:

```text
evidence/
├── 01-clean-image-generation.png
├── 02-lsb-embedding.png
├── 03-lsb-pixel-verification.png
├── 04-sha256-reproducibility.png
└── 05-ground-truth-validation.png
```

Each screenshot is intended to prove a distinct investigation stage rather than repeat the same information.

The supporting Dataset B artifacts include:

```text
clean/
└── SS-001-CLEAN.png

modified/
└── SS-001-LSB-RAW.png

scripts/
├── generate_clean.py
└── embed_lsb.py

evidence/
├── ground-truth-manifest.csv
├── SS-001-CLEAN.sha256
├── SS-001-LSB-RAW.sha256
└── SS-001-reproducibility.sha256
```

The reproduced PNG files are verification artifacts, not additional independent test samples.

---

## Next Investigation

**Day 3: StegaShield Installation and Clean Baseline Testing**

The next stage will establish the StegaShield deployment and test known-clean images.

I will record the detector's actual predictions and probability scores, preserve the relevant evidence, and compare its output against independently established ground truth.

The goal is to begin evaluating detector behavior without confusing a model signal with proof of malicious activity.
