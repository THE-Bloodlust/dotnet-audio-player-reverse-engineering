# dotnet-audio-player-reverse-engineering

An intermediate-level reverse-engineering exercise to get comfortable with analyzing and modifying compiled .NET binaries.

# Reverse Engineering a .NET Audio Player to Unlock Higher Playback Speeds

## Overview

This project explores the reverse engineering of **ALVAS.AUDIO** specifically AudioCS.exe, a Windows Forms (.NET) audio player, to identify and modify its maximum playback-speed limit.

The objective was to understand how the application handled playback speed, trace the relationship between its frontend controls and backend audio engine, and modify the executable to support higher playback speeds.

Rather than modifying the original source code, the investigation was performed directly on the compiled executable using .NET decompilation and assembly-patching tools.

### Objectives

- Analyze a compiled Windows Forms application.
- Inspect and understand its decompiled C# code.
- Trace UI events to their corresponding backend functionality.
- Identify the mechanism responsible for restricting playback speed.
- Modify the executable to increase the maximum playback speed.

### Tools Used

| Tool | Purpose |
|---|---|
| JetBrains dotPeek | Static analysis and C# decompilation |
| dnSpy | Editing methods and patching the executable |
| Windows Forms | Understanding the application's UI controls |
| Windows Multimedia API | Understanding backend playback-speed control |

---

## 1. Bypassing the Decompilation Restriction

The first step was loading the compiled executable into JetBrains dotPeek.

However, the application's `Player` constructor initially appeared as a decompilation warning:

```csharp
public Player()
{
    // This assembly is protected from decompilation.
    // Enable "Decompile protected assemblies
    // (ignore SuppressIldasm attribute)".
}
```

This indicated that the assembly contained the `SuppressIldasm` attribute, which discourages decompilation by compatible tools.

To inspect the code, I simply changed the decompiler options:

1. Opened **Tools → Options** in dotPeek.
2. Navigated to the Decompiler settings.
3. Enabled **Decompile protected assemblies (ignore SuppressIldasm attribute)**.
4. Reloaded the executable.

After this, dotPeek was able to display the application's decompiled C# code.

---

## 2. Identifying the Audio Engine and UI Controls

During the analysis, two important components were identified.

### 2.1 Backend Playback-Speed Property

The `Player` class contained a `Speed` property responsible for reading and modifying playback speed.

```csharp
public int Speed
{
    get
    {
        try
        {
            int capacity = 128;
            StringBuilder stringBuilder =
                new StringBuilder(capacity);

            B.A(.aC(), stringBuilder, capacity, 0);

            return int.Parse(stringBuilder.ToString());
        }
        catch
        {
            return -1;
        }
    }

    set => B.A(
        .ac() + (object)value,
        (StringBuilder)null,
        0,
        0
    );
}
```

The implementation used an obfuscated wrapper around the Windows Multimedia API.

The underlying `mciSendString` interface was responsible for communicating playback-speed commands to the audio engine. I did not investigate the audio engine's internal implementation further, as the main focus of this exercise was the application's playback-speed restriction.

An important observation was the speed representation:

- `1000` = 1.0x playback speed.
- `2000` = 2.0x playback speed.
- `3000` = 3.0x playback speed.
- `6000` = 6.0x playback speed.

This established how the application represented playback speed internally.

### 2.2 Frontend Speed Slider

The application's main form contained a TrackBar named `tbSpeed`.

Its initialization included:

```csharp
this.tbSpeed.Name = "tbSpeed";
this.tbSpeed.Orientation = Orientation.Vertical;
this.tbSpeed.Value = 5;
this.tbSpeed.Scroll +=
    new EventHandler(this.tbSpeed_Scroll);
```

An interesting detail was that no explicit `Maximum` property was assigned.

This became important during the investigation.

---

## 3. Tracing the Playback-Speed Logic

Initially, searching for an explicit maximum-speed variable or a hardcoded playback-speed restriction did not reveal anything useful.

Inspecting resource files and following references to framework properties also led to unrelated Windows Forms implementation details.

Instead of continuing to search for a maximum-speed variable, I followed the event registration:

```csharp
this.tbSpeed.Scroll +=
    new EventHandler(this.tbSpeed_Scroll);
```

This led directly to the application's `tbSpeed_Scroll` event handler.

### 3.1 The Event Handler

```csharp
private void tbSpeed_Scroll(object sender, EventArgs e)
{
    if (this.tbSpeed.Value == 5)
        this.pl.Speed = 1000;

    else if (this.tbSpeed.Value > 5)
        this.pl.Speed =
            1000 * (this.tbSpeed.Value - 5 + 1);

    else if (this.tbSpeed.Value < 5)
        this.pl.Speed =
            1000 / (5 - this.tbSpeed.Value);

    this.lblSpeed.Text =
        $"Speed: {this.pl.Speed / 10}";
}
```

The event handler revealed the actual playback-speed calculation.

The slider value determined the speed passed to the backend.

### 3.2 Understanding the Default Limit

The TrackBar did not have an explicitly assigned `Maximum` value.

Windows Forms TrackBar controls have a default maximum value of `10`.

The slider was initialized at value `5`, which represented normal playback speed.

The calculation worked as follows:

| Slider Value | Backend Speed | Playback Speed |
|---|---:|---:|
| 5 | 1000 | 1x |
| 6 | 2000 | 2x |
| 7 | 3000 | 3x |
| 8 | 4000 | 4x |
| 9 | 5000 | 5x |
| 10 | 6000 | 6x |

This explained the original limitation.

The event handler itself did not contain a hardcoded 6x maximum. Instead, the restriction resulted from the TrackBar's default maximum value.

Increasing the slider's maximum would therefore allow the existing calculation to produce higher playback speeds without changing the event handler.

---

## 4. Patching the Executable Using dnSpy

Since dotPeek was used only for analysis, dnSpy was required to modify the compiled application.

The modification involved extending the TrackBar's maximum value.

### 4.1 Locating the Initialization Method

1. Opened the executable in dnSpy.
2. Navigated to the `MainForm` class.
3. Located `InitializeComponent()`.
4. Selected **Edit Method (C#)**.

### 4.2 Modifying the TrackBar Maximum

The following property assignment was added to the TrackBar initialization:

```csharp
this.tbSpeed.Maximum = 14;
```

The relevant initialization became:

```csharp
this.tbSpeed.Name = "tbSpeed";
this.tbSpeed.Orientation = Orientation.Vertical;
this.tbSpeed.Value = 5;
this.tbSpeed.Maximum = 14;

this.tbSpeed.Scroll +=
    new EventHandler(this.tbSpeed_Scroll);
```

The original event handler was left unchanged.

### 4.3 Expected Result

With the maximum value increased from `10` to `14`, the existing formula produces:

| Slider Value | Backend Speed | Playback Speed |
|---|---:|---:|
| 10 | 6000 | 6x |
| 11 | 7000 | 7x |
| 12 | 8000 | 8x |
| 13 | 9000 | 9x |
| 14 | 10000 | 10x |

The modification extends the available playback-speed range from 6x to 10x.

---

---

## 6. Key Takeaways

This investigation demonstrated several useful reverse-engineering techniques:

- **Decompilation:** Understanding compiled .NET applications through reconstructed C# source code.
- **Event tracing:** Following UI event registrations to locate the underlying application logic.
- **Framework knowledge:** Recognizing how default Windows Forms properties can influence application behavior.
- **Backend analysis:** Understanding how frontend controls interact with an underlying audio engine.
- **Binary patching:** Modifying application behavior without access to its original source code.

One of the most interesting findings was that the playback-speed restriction was not implemented through an explicit condition in the application's logic. It was an indirect consequence of a framework default.

This was a useful reminder that reverse engineering is not always about finding a specific hardcoded value. Sometimes, understanding the interaction between application logic and framework behavior is what reveals the actual limitation.

---

## 7. Repository Contents

If you want to try the original audio player yourself, I've included it in a separate folder inside this repository.

```text
dotnet-audio-player-reverse-engineering/
│
├── README.md
│
└── bin/
    └── AudioCS.exe
```

The `bin/` folder contains the original AudioCS.exe executable used during this exercise.

You can use it to follow the analysis, explore the application's internals, and reproduce the reverse-engineering process.

---

**Disclaimer:** This project was conducted for educational purposes to explore .NET application internals, static analysis, and executable modification. Please respect the original application's licensing and redistribution terms when using or sharing the included executable.
