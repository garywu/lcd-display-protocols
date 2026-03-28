# LCD Display Protocols & Multi-Color LED Matrix Libraries — HD44780 to RGB LED Panels

Building a virtual LCD display means understanding the protocols that real displays speak. Whether you're emulating an HD44780 character LCD in a macOS notch HUD, rendering RGB pixels on a canvas, or accepting commands from existing embedded libraries — the protocols and graphics primitives are the foundation.

This article covers the full stack: from the venerable HD44780 command set through modern RGB LED matrix libraries, color display controllers, graphics primitive APIs, network protocols for remote displays, and practical TypeScript implementations for virtual display engines.

**What you'll learn:**

- The complete HD44780 instruction set and how to implement a virtual HD44780 in TypeScript
- How color display controllers (SSD1306, SSD1331, ST7735, ILI9341) structure their command protocols
- The Adafruit GFX graphics primitive API and how to port it to Canvas
- RGB LED matrix libraries: FastLED, SmartMatrix, HUB75, WLED, Pixelblaze
- Existing LED matrix simulators and virtual LCD projects
- How to build a protocol adapter that accepts standard LCD commands over WebSocket
- Color formats (RGB565, RGB888, HSV) and conversion between them
- Practical multi-color display patterns: scrolling text, bar charts, sprites, dashboards

---

## The Problem

You've built a dot-matrix display renderer — a Canvas element drawing 5x7 pixel characters in a grid. It works. But it's bespoke. Every feature you add (cursor blinking, display shifting, custom characters) is ad-hoc code with no standard behavior to reference.

Meanwhile, the embedded world has decades of established display protocols. The HD44780 has been the standard character LCD controller since the 1980s. Color displays like the ILI9341 and SSD1331 have well-documented command sets. Libraries like Adafruit GFX provide a universal graphics primitive API that works across hundreds of display types.

The gap: these protocols exist for hardware. Nobody has assembled a clear reference for implementing them in software — particularly for a Canvas-based virtual display on macOS.

What changes if you get this right:

- **Compatibility**: existing LCD client libraries can drive your virtual display
- **Completeness**: the HD44780 spec tells you exactly what cursor behavior, display shifting, and CGRAM custom characters should do
- **Multi-color support**: color display protocols give you a roadmap for going beyond monochrome
- **Network control**: protocol adapters let remote processes push content to your display

---

## Core Concepts

### 1. The HD44780 Command Architecture

The [Hitachi HD44780](https://en.wikipedia.org/wiki/Hitachi_HD44780_LCD_controller) is the most widely cloned LCD controller in history. Every 16x2 or 20x4 character LCD you've seen likely speaks HD44780. Understanding its command set is understanding the lingua franca of character displays.

The HD44780 has two registers accessed via the RS (Register Select) pin:

```typescript
interface HD44780Registers {
  // RS=0: Instruction Register — accepts commands
  // RS=1: Data Register — accepts character data for DDRAM/CGRAM

  instructionRegister: number; // Write commands here
  dataRegister: number;       // Write display data here
}

interface HD44780State {
  // Display Data RAM — 80 bytes (what's shown on screen)
  ddram: Uint8Array;          // 80 bytes: row1[0..39], row2[64..103]
  ddramAddress: number;       // Current write position (0-79 or 64-103)

  // Character Generator RAM — 64 bytes (custom characters)
  cgram: Uint8Array;          // 64 bytes: 8 characters × 8 rows
  cgramAddress: number;       // Current CGRAM write position

  // Cursor position
  cursorRow: number;
  cursorCol: number;

  // Display control flags
  displayOn: boolean;
  cursorOn: boolean;
  cursorBlink: boolean;

  // Entry mode flags
  incrementMode: boolean;     // true=increment, false=decrement
  displayShift: boolean;      // true=shift display on write

  // Function set
  eightBitMode: boolean;      // true=8-bit, false=4-bit
  twoLineMode: boolean;       // true=2 lines, false=1 line
  largeFont: boolean;         // true=5x10, false=5x8

  // Display shift offset
  shiftOffset: number;

  // Internal
  busyFlag: boolean;
}
```

> **Key insight:** The HD44780 is fundamentally a state machine. Every command modifies internal state (cursor position, display flags, RAM addresses). A virtual HD44780 is just this state machine plus a renderer that reads DDRAM to produce pixels.

### 2. Instruction Decoding

The HD44780 uses a priority-encoded instruction format — the highest set bit determines the command:

```typescript
enum HD44780Command {
  // Bit patterns (RS=0, RW=0 for all writes)
  CLEAR_DISPLAY       = 0x01, // 00000001
  RETURN_HOME         = 0x02, // 0000001x
  ENTRY_MODE_SET      = 0x04, // 000001xx — bits: I/D, S
  DISPLAY_CONTROL     = 0x08, // 00001xxx — bits: D, C, B
  CURSOR_SHIFT        = 0x10, // 0001xxxx — bits: S/C, R/L
  FUNCTION_SET        = 0x20, // 001xxxxx — bits: DL, N, F
  SET_CGRAM_ADDR      = 0x40, // 01xxxxxx — 6-bit address
  SET_DDRAM_ADDR      = 0x80, // 1xxxxxxx — 7-bit address
}

// Decode an instruction byte
function decodeInstruction(byte: number): {
  command: string;
  params: Record<string, boolean | number>;
} {
  if (byte & 0x80) {
    return {
      command: 'SET_DDRAM_ADDR',
      params: { address: byte & 0x7F }
    };
  }
  if (byte & 0x40) {
    return {
      command: 'SET_CGRAM_ADDR',
      params: { address: byte & 0x3F }
    };
  }
  if (byte & 0x20) {
    return {
      command: 'FUNCTION_SET',
      params: {
        eightBit: !!(byte & 0x10),  // DL
        twoLine:  !!(byte & 0x08),  // N
        largeFont: !!(byte & 0x04), // F
      }
    };
  }
  if (byte & 0x10) {
    return {
      command: 'CURSOR_SHIFT',
      params: {
        shiftDisplay: !!(byte & 0x08), // S/C: 1=display, 0=cursor
        shiftRight:   !!(byte & 0x04), // R/L: 1=right, 0=left
      }
    };
  }
  if (byte & 0x08) {
    return {
      command: 'DISPLAY_CONTROL',
      params: {
        displayOn:   !!(byte & 0x04), // D
        cursorOn:    !!(byte & 0x02), // C
        cursorBlink: !!(byte & 0x01), // B
      }
    };
  }
  if (byte & 0x04) {
    return {
      command: 'ENTRY_MODE_SET',
      params: {
        increment:    !!(byte & 0x02), // I/D: 1=increment, 0=decrement
        displayShift: !!(byte & 0x01), // S: 1=shift display on write
      }
    };
  }
  if (byte & 0x02) {
    return { command: 'RETURN_HOME', params: {} };
  }
  if (byte & 0x01) {
    return { command: 'CLEAR_DISPLAY', params: {} };
  }
  return { command: 'NOP', params: {} };
}
```

### 3. DDRAM Address Mapping

The HD44780's DDRAM layout is notoriously non-contiguous. For a 2-line display, row 2 doesn't start at address 40 — it starts at address 64 (0x40):

```typescript
// DDRAM layout for common display sizes
const DDRAM_ROW_OFFSETS: Record<string, number[]> = {
  '16x2': [0x00, 0x40],
  '20x4': [0x00, 0x40, 0x14, 0x54],  // Row 3 wraps after row 1!
  '40x2': [0x00, 0x40],
  '20x2': [0x00, 0x40],
  '16x1': [0x00],                      // Some 16x1s are actually 8x2
  '8x2':  [0x00, 0x40],
};

// Convert DDRAM address to row/col
function ddramToPosition(
  address: number,
  cols: number,
  rows: number
): { row: number; col: number } | null {
  const offsets = rows === 4
    ? [0x00, 0x40, 0x00 + cols, 0x40 + cols]
    : [0x00, 0x40];

  for (let row = 0; row < offsets.length; row++) {
    const start = offsets[row];
    if (address >= start && address < start + cols) {
      return { row, col: address - start };
    }
  }
  return null;
}

// Convert row/col to DDRAM address
function positionToDdram(
  row: number,
  col: number,
  cols: number
): number {
  const rowOffsets = [0x00, 0x40, cols, 0x40 + cols];
  return rowOffsets[row] + col;
}
```

> **Key insight:** The 20x4 display layout is especially tricky — row 3 (index 2) continues from the end of row 1's DDRAM space, and row 4 (index 3) continues from row 2. This means scrolling a 20x4 display doesn't work like you'd expect.

### 4. Color Formats

Moving beyond monochrome, displays use different color encodings. The three you need to know:

```typescript
// RGB565: 16-bit color (5 red, 6 green, 5 blue)
// Used by: ST7735, ILI9341, Adafruit GFX, most TFT displays
type RGB565 = number; // uint16

// RGB888: 24-bit true color (8 bits per channel)
// Used by: Canvas, CSS, most software rendering
type RGB888 = { r: number; g: number; b: number };

// HSV: Hue-Saturation-Value
// Used by: FastLED, WLED, color animations
type HSV = { h: number; s: number; v: number }; // h: 0-360, s/v: 0-100

function rgb888ToRgb565(r: number, g: number, b: number): RGB565 {
  return ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
}

function rgb565ToRgb888(color: RGB565): RGB888 {
  const r = ((color >> 11) & 0x1F) << 3;
  const g = ((color >> 5) & 0x3F) << 2;
  const b = (color & 0x1F) << 3;
  // Fill lower bits for full range
  return {
    r: r | (r >> 5),
    g: g | (g >> 6),
    b: b | (b >> 5),
  };
}

function rgb888ToHsv(r: number, g: number, b: number): HSV {
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b);
  const min = Math.min(r, g, b);
  const d = max - min;

  let h = 0;
  const s = max === 0 ? 0 : d / max;
  const v = max;

  if (d !== 0) {
    switch (max) {
      case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break;
      case g: h = ((b - r) / d + 2) / 6; break;
      case b: h = ((r - g) / d + 4) / 6; break;
    }
  }

  return { h: h * 360, s: s * 100, v: v * 100 };
}

function hsvToRgb888(h: number, s: number, v: number): RGB888 {
  h /= 360; s /= 100; v /= 100;
  const i = Math.floor(h * 6);
  const f = h * 6 - i;
  const p = v * (1 - s);
  const q = v * (1 - f * s);
  const t = v * (1 - (1 - f) * s);

  let r: number, g: number, b: number;
  switch (i % 6) {
    case 0: r = v; g = t; b = p; break;
    case 1: r = q; g = v; b = p; break;
    case 2: r = p; g = v; b = t; break;
    case 3: r = p; g = q; b = v; break;
    case 4: r = t; g = p; b = v; break;
    case 5: r = v; g = p; b = q; break;
    default: r = 0; g = 0; b = 0;
  }

  return {
    r: Math.round(r * 255),
    g: Math.round(g * 255),
    b: Math.round(b * 255),
  };
}

// Named color constants (Adafruit GFX convention, RGB565)
const COLOR = {
  BLACK:   0x0000,
  WHITE:   0xFFFF,
  RED:     0xF800,
  GREEN:   0x07E0,
  BLUE:    0x001F,
  CYAN:    0x07FF,
  MAGENTA: 0xF81F,
  YELLOW:  0xFFE0,
  ORANGE:  0xFD20,
} as const;
```

> **Key insight:** RGB565 gives green one extra bit because the human eye is most sensitive to green wavelengths. This isn't arbitrary — it's perceptual optimization in 16 bits. When converting RGB565 back to RGB888, you must fill the lower bits (not just shift) or you'll never reach full white (255,255,255).

### 5. The Graphics Primitive Interface

The [Adafruit GFX library](https://github.com/adafruit/Adafruit-GFX-Library) established the standard API for drawing on small displays. Every display library in the Arduino ecosystem derives from it. Here's the interface in TypeScript:

```typescript
interface GFXDisplay {
  readonly width: number;
  readonly height: number;

  // The one function every display MUST implement
  drawPixel(x: number, y: number, color: number): void;

  // Everything else is built on drawPixel
  drawLine(x0: number, y0: number, x1: number, y1: number, color: number): void;
  drawFastHLine(x: number, y: number, w: number, color: number): void;
  drawFastVLine(x: number, y: number, h: number, color: number): void;

  drawRect(x: number, y: number, w: number, h: number, color: number): void;
  fillRect(x: number, y: number, w: number, h: number, color: number): void;

  drawCircle(x0: number, y0: number, r: number, color: number): void;
  fillCircle(x0: number, y0: number, r: number, color: number): void;

  drawTriangle(
    x0: number, y0: number,
    x1: number, y1: number,
    x2: number, y2: number,
    color: number
  ): void;
  fillTriangle(
    x0: number, y0: number,
    x1: number, y1: number,
    x2: number, y2: number,
    color: number
  ): void;

  drawRoundRect(x: number, y: number, w: number, h: number, radius: number, color: number): void;
  fillRoundRect(x: number, y: number, w: number, h: number, radius: number, color: number): void;

  drawBitmap(x: number, y: number, bitmap: Uint8Array, w: number, h: number, color: number): void;
  drawChar(x: number, y: number, c: string, color: number, bg: number, size: number): void;

  fillScreen(color: number): void;
  setRotation(r: number): void; // 0-3 for 0/90/180/270 degrees

  // Text cursor
  setCursor(x: number, y: number): void;
  setTextColor(fg: number, bg?: number): void;
  setTextSize(s: number): void;
  setTextWrap(wrap: boolean): void;
  print(text: string): void;
  println(text: string): void;

  // Color helper
  color565(r: number, g: number, b: number): number;
}
```

---

## Patterns

### Pattern 1: Virtual HD44780 Controller

A complete virtual HD44780 that accepts the same byte commands as the real chip. This is the core state machine — pair it with any renderer (Canvas, SwiftUI, terminal).

```typescript
type HD44780EventType =
  | 'clear'
  | 'home'
  | 'cursor-move'
  | 'display-update'
  | 'display-control'
  | 'shift'
  | 'cgram-update'
  | 'function-set';

type HD44780Event = {
  type: HD44780EventType;
  state: Readonly<VirtualHD44780>;
};

type HD44780Listener = (event: HD44780Event) => void;

class VirtualHD44780 {
  // Memory
  readonly ddram = new Uint8Array(128).fill(0x20);  // Space-filled
  readonly cgram = new Uint8Array(64).fill(0x00);

  // Configuration
  cols: number;
  rows: number;

  // State
  ddramAddress = 0x00;
  cgramAddress = 0x00;
  addressingCgram = false;

  displayOn = true;
  cursorOn = false;
  cursorBlink = false;

  incrementMode = true;
  displayShift = false;

  eightBitMode = true;
  twoLineMode = true;
  largeFont = false;

  shiftOffset = 0;

  private listeners: HD44780Listener[] = [];
  private nibbleBuffer: number | null = null;

  constructor(cols = 16, rows = 2) {
    this.cols = cols;
    this.rows = rows;
  }

  onEvent(listener: HD44780Listener): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  private emit(type: HD44780EventType): void {
    const event: HD44780Event = { type, state: this };
    for (const listener of this.listeners) {
      listener(event);
    }
  }

  // RS=0: Write instruction
  writeInstruction(byte: number): void {
    if (!this.eightBitMode) {
      // 4-bit mode: accumulate two nibbles
      if (this.nibbleBuffer === null) {
        this.nibbleBuffer = (byte & 0xF0);
        return;
      }
      byte = this.nibbleBuffer | ((byte >> 4) & 0x0F);
      this.nibbleBuffer = null;
    }

    this.executeInstruction(byte);
  }

  // RS=1: Write data
  writeData(byte: number): void {
    if (!this.eightBitMode) {
      if (this.nibbleBuffer === null) {
        this.nibbleBuffer = (byte & 0xF0);
        return;
      }
      byte = this.nibbleBuffer | ((byte >> 4) & 0x0F);
      this.nibbleBuffer = null;
    }

    if (this.addressingCgram) {
      this.cgram[this.cgramAddress & 0x3F] = byte;
      this.cgramAddress = (this.cgramAddress + 1) & 0x3F;
      this.emit('cgram-update');
    } else {
      this.ddram[this.ddramAddress & 0x7F] = byte;
      this.advanceCursor();
      this.emit('display-update');
    }
  }

  private executeInstruction(byte: number): void {
    if (byte & 0x80) {
      // Set DDRAM Address
      this.ddramAddress = byte & 0x7F;
      this.addressingCgram = false;
      this.emit('cursor-move');
      return;
    }

    if (byte & 0x40) {
      // Set CGRAM Address
      this.cgramAddress = byte & 0x3F;
      this.addressingCgram = true;
      this.emit('cursor-move');
      return;
    }

    if (byte & 0x20) {
      // Function Set
      this.eightBitMode = !!(byte & 0x10);
      this.twoLineMode = !!(byte & 0x08);
      this.largeFont = !!(byte & 0x04);
      this.emit('function-set');
      return;
    }

    if (byte & 0x10) {
      // Cursor or Display Shift
      const shiftDisplay = !!(byte & 0x08);
      const shiftRight = !!(byte & 0x04);

      if (shiftDisplay) {
        this.shiftOffset += shiftRight ? 1 : -1;
      } else {
        if (shiftRight) {
          this.ddramAddress = (this.ddramAddress + 1) & 0x7F;
        } else {
          this.ddramAddress = (this.ddramAddress - 1) & 0x7F;
        }
      }
      this.emit('shift');
      return;
    }

    if (byte & 0x08) {
      // Display On/Off Control
      this.displayOn = !!(byte & 0x04);
      this.cursorOn = !!(byte & 0x02);
      this.cursorBlink = !!(byte & 0x01);
      this.emit('display-control');
      return;
    }

    if (byte & 0x04) {
      // Entry Mode Set
      this.incrementMode = !!(byte & 0x02);
      this.displayShift = !!(byte & 0x01);
      this.emit('display-control');
      return;
    }

    if (byte & 0x02) {
      // Return Home
      this.ddramAddress = 0x00;
      this.shiftOffset = 0;
      this.emit('home');
      return;
    }

    if (byte & 0x01) {
      // Clear Display
      this.ddram.fill(0x20);
      this.ddramAddress = 0x00;
      this.shiftOffset = 0;
      this.incrementMode = true;
      this.emit('clear');
      return;
    }
  }

  private advanceCursor(): void {
    if (this.incrementMode) {
      this.ddramAddress++;
    } else {
      this.ddramAddress--;
    }
    this.ddramAddress &= 0x7F;

    if (this.displayShift) {
      this.shiftOffset += this.incrementMode ? 1 : -1;
    }
  }

  // Convenience: get the visible characters for each row
  getVisibleText(): string[] {
    const result: string[] = [];
    const rowOffsets = this.rows === 4
      ? [0x00, 0x40, this.cols, 0x40 + this.cols]
      : this.rows === 1
        ? [0x00]
        : [0x00, 0x40];

    for (let row = 0; row < this.rows; row++) {
      let text = '';
      for (let col = 0; col < this.cols; col++) {
        const addr = (rowOffsets[row] + col + this.shiftOffset) & 0x7F;
        const charCode = this.ddram[addr];
        // Characters 0-7 are CGRAM custom characters
        if (charCode < 8) {
          text += `\x00`; // Placeholder — renderer handles CGRAM
        } else {
          text += String.fromCharCode(charCode);
        }
      }
      result.push(text);
    }
    return result;
  }

  // Get CGRAM character bitmap (8 rows of 5 bits each)
  getCustomCharacter(index: number): Uint8Array {
    const offset = (index & 0x07) * 8;
    return this.cgram.slice(offset, offset + 8);
  }

  // Get cursor position as row/col
  getCursorPosition(): { row: number; col: number } {
    const rowOffsets = this.rows === 4
      ? [0x00, 0x40, this.cols, 0x40 + this.cols]
      : [0x00, 0x40];

    for (let row = 0; row < rowOffsets.length; row++) {
      const start = rowOffsets[row];
      if (this.ddramAddress >= start && this.ddramAddress < start + this.cols) {
        return { row, col: this.ddramAddress - start };
      }
    }
    return { row: 0, col: this.ddramAddress };
  }

  // Helper: write a string at the current position
  printString(text: string): void {
    for (const char of text) {
      this.writeData(char.charCodeAt(0));
    }
  }

  // Helper: set cursor to row/col
  setCursor(row: number, col: number): void {
    const rowOffsets = [0x00, 0x40, this.cols, 0x40 + this.cols];
    this.writeInstruction(0x80 | (rowOffsets[row] + col));
  }

  // Helper: define a custom character (index 0-7)
  defineCharacter(index: number, bitmap: number[]): void {
    this.writeInstruction(0x40 | ((index & 0x07) << 3));
    for (const row of bitmap) {
      this.writeData(row & 0x1F);
    }
    // Restore DDRAM addressing
    this.addressingCgram = false;
  }
}
```

**Usage — driving the virtual HD44780 like hardware code would:**

```typescript
const lcd = new VirtualHD44780(16, 2);

// Standard initialization sequence (matches real hardware)
lcd.writeInstruction(0x38); // Function Set: 8-bit, 2-line, 5x8 font
lcd.writeInstruction(0x0C); // Display On, Cursor Off, Blink Off
lcd.writeInstruction(0x06); // Entry Mode: Increment, No Shift
lcd.writeInstruction(0x01); // Clear Display

// Write "Hello, World!" on line 1
lcd.printString('Hello, World!');

// Move to line 2, column 0
lcd.setCursor(1, 0);
lcd.printString('HD44780 Virtual');

// Define a custom heart character at index 0
lcd.defineCharacter(0, [
  0b00000,
  0b01010,
  0b11111,
  0b11111,
  0b11111,
  0b01110,
  0b00100,
  0b00000,
]);

// Write the custom character
lcd.setCursor(0, 15);
lcd.writeData(0x00); // Character 0 = our heart
```

### Pattern 2: Canvas-Based Dot Matrix Renderer

A renderer that takes a virtual HD44780 (or any pixel buffer) and draws it on an HTML Canvas with authentic LCD aesthetics — pixel gaps, glow effects, and configurable colors.

```typescript
interface DotMatrixTheme {
  backgroundColor: string;
  pixelOnColor: string;
  pixelOffColor: string;
  pixelGap: number;         // Gap between dots in pixels
  pixelRadius: number;      // 0=square, >0=rounded
  glowRadius: number;       // 0=no glow, >0=blur radius
  glowColor: string;
  dotShape: 'square' | 'circle';
}

const THEMES: Record<string, DotMatrixTheme> = {
  classicGreen: {
    backgroundColor: '#0a1a0a',
    pixelOnColor: '#33ff33',
    pixelOffColor: '#0a2a0a',
    pixelGap: 1,
    pixelRadius: 0,
    glowRadius: 2,
    glowColor: 'rgba(51, 255, 51, 0.3)',
    dotShape: 'square',
  },
  amberLCD: {
    backgroundColor: '#1a0f00',
    pixelOnColor: '#ffaa00',
    pixelOffColor: '#1a1200',
    pixelGap: 1,
    pixelRadius: 0,
    glowRadius: 2,
    glowColor: 'rgba(255, 170, 0, 0.3)',
    dotShape: 'square',
  },
  blueLED: {
    backgroundColor: '#000a1a',
    pixelOnColor: '#3399ff',
    pixelOffColor: '#001022',
    pixelGap: 1,
    pixelRadius: 1,
    glowRadius: 3,
    glowColor: 'rgba(51, 153, 255, 0.4)',
    dotShape: 'circle',
  },
  rgbMatrix: {
    backgroundColor: '#111111',
    pixelOnColor: '#ffffff',  // Overridden per-pixel for RGB
    pixelOffColor: '#1a1a1a',
    pixelGap: 1,
    pixelRadius: 1,
    glowRadius: 1,
    glowColor: 'rgba(255, 255, 255, 0.2)',
    dotShape: 'circle',
  },
  whiteOnBlue: {
    backgroundColor: '#0000aa',
    pixelOnColor: '#ffffff',
    pixelOffColor: '#0000cc',
    pixelGap: 1,
    pixelRadius: 0,
    glowRadius: 0,
    glowColor: 'transparent',
    dotShape: 'square',
  },
};

class DotMatrixRenderer {
  private ctx: CanvasRenderingContext2D;
  private pixelSize: number;
  private theme: DotMatrixTheme;

  constructor(
    private canvas: HTMLCanvasElement,
    private gridWidth: number,   // Total pixel columns
    private gridHeight: number,  // Total pixel rows
    theme: DotMatrixTheme = THEMES.classicGreen,
  ) {
    this.ctx = canvas.getContext('2d')!;
    this.theme = theme;

    // Calculate pixel size to fill canvas
    const cellWidth = canvas.width / gridWidth;
    const cellHeight = canvas.height / gridHeight;
    this.pixelSize = Math.min(cellWidth, cellHeight);
  }

  setTheme(theme: DotMatrixTheme): void {
    this.theme = theme;
  }

  // Render a monochrome pixel buffer (1 = on, 0 = off)
  renderMono(pixels: Uint8Array): void {
    const { ctx, theme, pixelSize, gridWidth, gridHeight } = this;
    const gap = theme.pixelGap;
    const dotSize = pixelSize - gap;

    ctx.fillStyle = theme.backgroundColor;
    ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    // Apply glow effect
    if (theme.glowRadius > 0) {
      ctx.save();
      ctx.shadowBlur = theme.glowRadius;
      ctx.shadowColor = theme.glowColor;
    }

    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        const on = pixels[y * gridWidth + x] !== 0;
        ctx.fillStyle = on ? theme.pixelOnColor : theme.pixelOffColor;

        const px = x * pixelSize + gap / 2;
        const py = y * pixelSize + gap / 2;

        if (theme.dotShape === 'circle') {
          ctx.beginPath();
          ctx.arc(px + dotSize / 2, py + dotSize / 2, dotSize / 2, 0, Math.PI * 2);
          ctx.fill();
        } else if (theme.pixelRadius > 0) {
          this.roundRect(px, py, dotSize, dotSize, theme.pixelRadius);
        } else {
          ctx.fillRect(px, py, dotSize, dotSize);
        }
      }
    }

    if (theme.glowRadius > 0) {
      ctx.restore();
    }
  }

  // Render an RGB pixel buffer (3 bytes per pixel: R, G, B)
  renderRGB(pixels: Uint8Array): void {
    const { ctx, theme, pixelSize, gridWidth, gridHeight } = this;
    const gap = theme.pixelGap;
    const dotSize = pixelSize - gap;

    ctx.fillStyle = theme.backgroundColor;
    ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        const i = (y * gridWidth + x) * 3;
        const r = pixels[i];
        const g = pixels[i + 1];
        const b = pixels[i + 2];

        const isOff = r === 0 && g === 0 && b === 0;

        if (isOff) {
          ctx.fillStyle = theme.pixelOffColor;
        } else {
          ctx.fillStyle = `rgb(${r}, ${g}, ${b})`;
          if (theme.glowRadius > 0) {
            ctx.shadowBlur = theme.glowRadius;
            ctx.shadowColor = `rgba(${r}, ${g}, ${b}, 0.4)`;
          }
        }

        const px = x * pixelSize + gap / 2;
        const py = y * pixelSize + gap / 2;

        if (theme.dotShape === 'circle') {
          ctx.beginPath();
          ctx.arc(px + dotSize / 2, py + dotSize / 2, dotSize / 2, 0, Math.PI * 2);
          ctx.fill();
        } else {
          ctx.fillRect(px, py, dotSize, dotSize);
        }

        ctx.shadowBlur = 0;
      }
    }
  }

  // Render an RGB565 pixel buffer (2 bytes per pixel, big-endian)
  renderRGB565(pixels: Uint16Array): void {
    const rgb = new Uint8Array(pixels.length * 3);
    for (let i = 0; i < pixels.length; i++) {
      const c = pixels[i];
      const { r, g, b } = rgb565ToRgb888(c);
      rgb[i * 3] = r;
      rgb[i * 3 + 1] = g;
      rgb[i * 3 + 2] = b;
    }
    this.renderRGB(rgb);
  }

  private roundRect(
    x: number, y: number, w: number, h: number, r: number
  ): void {
    const ctx = this.ctx;
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + w - r, y);
    ctx.quadraticCurveTo(x + w, y, x + w, y + r);
    ctx.lineTo(x + w, y + h - r);
    ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
    ctx.lineTo(x + r, y + h);
    ctx.quadraticCurveTo(x, y + h, x, y + h - r);
    ctx.lineTo(x, y + r);
    ctx.quadraticCurveTo(x, y, x + r, y);
    ctx.fill();
  }
}
```

### Pattern 3: GFX Graphics Primitives on a Pixel Buffer

A TypeScript port of the [Adafruit GFX](https://learn.adafruit.com/adafruit-gfx-graphics-library/graphics-primitives) drawing primitives. This operates on an abstract pixel buffer — connect it to a Canvas renderer or any other output.

```typescript
// Standard 5x7 font (ASCII 32-126), packed as 5 bytes per character
// Each byte is a column, LSB is top row
const FONT_5X7: Uint8Array = new Uint8Array([
  // Space (0x20)
  0x00, 0x00, 0x00, 0x00, 0x00,
  // ! (0x21)
  0x00, 0x00, 0x5F, 0x00, 0x00,
  // " (0x22)
  0x00, 0x07, 0x00, 0x07, 0x00,
  // # (0x23)
  0x14, 0x7F, 0x14, 0x7F, 0x14,
  // $ (0x24)
  0x24, 0x2A, 0x7F, 0x2A, 0x12,
  // % (0x25)
  0x23, 0x13, 0x08, 0x64, 0x62,
  // & (0x26)
  0x36, 0x49, 0x55, 0x22, 0x50,
  // ' (0x27)
  0x00, 0x05, 0x03, 0x00, 0x00,
  // ( (0x28)
  0x00, 0x1C, 0x22, 0x41, 0x00,
  // ) (0x29)
  0x00, 0x41, 0x22, 0x1C, 0x00,
  // * (0x2A)
  0x08, 0x2A, 0x1C, 0x2A, 0x08,
  // + (0x2B)
  0x08, 0x08, 0x3E, 0x08, 0x08,
  // , (0x2C)
  0x00, 0x50, 0x30, 0x00, 0x00,
  // - (0x2D)
  0x08, 0x08, 0x08, 0x08, 0x08,
  // . (0x2E)
  0x00, 0x60, 0x60, 0x00, 0x00,
  // / (0x2F)
  0x20, 0x10, 0x08, 0x04, 0x02,
  // 0 (0x30)
  0x3E, 0x51, 0x49, 0x45, 0x3E,
  // 1 (0x31)
  0x00, 0x42, 0x7F, 0x40, 0x00,
  // 2 (0x32)
  0x42, 0x61, 0x51, 0x49, 0x46,
  // 3 (0x33)
  0x21, 0x41, 0x45, 0x4B, 0x31,
  // 4 (0x34)
  0x18, 0x14, 0x12, 0x7F, 0x10,
  // 5 (0x35)
  0x27, 0x45, 0x45, 0x45, 0x39,
  // 6 (0x36)
  0x3C, 0x4A, 0x49, 0x49, 0x30,
  // 7 (0x37)
  0x01, 0x71, 0x09, 0x05, 0x03,
  // 8 (0x38)
  0x36, 0x49, 0x49, 0x49, 0x36,
  // 9 (0x39)
  0x06, 0x49, 0x49, 0x29, 0x1E,
  // : (0x3A)
  0x00, 0x36, 0x36, 0x00, 0x00,
  // ; (0x3B)
  0x00, 0x56, 0x36, 0x00, 0x00,
  // < (0x3C)
  0x00, 0x08, 0x14, 0x22, 0x41,
  // = (0x3D)
  0x14, 0x14, 0x14, 0x14, 0x14,
  // > (0x3E)
  0x41, 0x22, 0x14, 0x08, 0x00,
  // ? (0x3F)
  0x02, 0x01, 0x51, 0x09, 0x06,
  // @ (0x40)
  0x32, 0x49, 0x79, 0x41, 0x3E,
  // A (0x41)
  0x7E, 0x11, 0x11, 0x11, 0x7E,
  // B (0x42)
  0x7F, 0x49, 0x49, 0x49, 0x36,
  // C (0x43)
  0x3E, 0x41, 0x41, 0x41, 0x22,
  // D (0x44)
  0x7F, 0x41, 0x41, 0x22, 0x1C,
  // E (0x45)
  0x7F, 0x49, 0x49, 0x49, 0x41,
  // F (0x46)
  0x7F, 0x09, 0x09, 0x01, 0x01,
  // G (0x47)
  0x3E, 0x41, 0x41, 0x51, 0x32,
  // H (0x48)
  0x7F, 0x08, 0x08, 0x08, 0x7F,
  // I (0x49)
  0x00, 0x41, 0x7F, 0x41, 0x00,
  // J (0x4A)
  0x20, 0x40, 0x41, 0x3F, 0x01,
  // K (0x4B)
  0x7F, 0x08, 0x14, 0x22, 0x41,
  // L (0x4C)
  0x7F, 0x40, 0x40, 0x40, 0x40,
  // M (0x4D)
  0x7F, 0x02, 0x04, 0x02, 0x7F,
  // N (0x4E)
  0x7F, 0x04, 0x08, 0x10, 0x7F,
  // O (0x4F)
  0x3E, 0x41, 0x41, 0x41, 0x3E,
  // P (0x50)
  0x7F, 0x09, 0x09, 0x09, 0x06,
  // Q (0x51)
  0x3E, 0x41, 0x51, 0x21, 0x5E,
  // R (0x52)
  0x7F, 0x09, 0x19, 0x29, 0x46,
  // S (0x53)
  0x46, 0x49, 0x49, 0x49, 0x31,
  // T (0x54)
  0x01, 0x01, 0x7F, 0x01, 0x01,
  // U (0x55)
  0x3F, 0x40, 0x40, 0x40, 0x3F,
  // V (0x56)
  0x1F, 0x20, 0x40, 0x20, 0x1F,
  // W (0x57)
  0x7F, 0x20, 0x18, 0x20, 0x7F,
  // X (0x58)
  0x63, 0x14, 0x08, 0x14, 0x63,
  // Y (0x59)
  0x03, 0x04, 0x78, 0x04, 0x03,
  // Z (0x5A)
  0x61, 0x51, 0x49, 0x45, 0x43,
]);

class PixelBuffer implements GFXDisplay {
  readonly buffer: Uint16Array; // RGB565 color per pixel
  private cursorX = 0;
  private cursorY = 0;
  private textColor: number = 0xFFFF;
  private textBg: number = 0x0000;
  private textSize = 1;
  private wrap = true;
  private rotation = 0;
  private _width: number;
  private _height: number;

  constructor(
    public readonly width: number,
    public readonly height: number,
  ) {
    this._width = width;
    this._height = height;
    this.buffer = new Uint16Array(width * height);
  }

  drawPixel(x: number, y: number, color: number): void {
    // Apply rotation
    [x, y] = this.applyRotation(x, y);
    if (x < 0 || x >= this.width || y < 0 || y >= this.height) return;
    this.buffer[y * this.width + x] = color;
  }

  private applyRotation(x: number, y: number): [number, number] {
    switch (this.rotation) {
      case 1: return [this.width - 1 - y, x];
      case 2: return [this.width - 1 - x, this.height - 1 - y];
      case 3: return [y, this.height - 1 - x];
      default: return [x, y];
    }
  }

  drawLine(x0: number, y0: number, x1: number, y1: number, color: number): void {
    // Bresenham's line algorithm
    const steep = Math.abs(y1 - y0) > Math.abs(x1 - x0);
    if (steep) {
      [x0, y0] = [y0, x0];
      [x1, y1] = [y1, x1];
    }
    if (x0 > x1) {
      [x0, x1] = [x1, x0];
      [y0, y1] = [y1, y0];
    }

    const dx = x1 - x0;
    const dy = Math.abs(y1 - y0);
    let err = dx / 2;
    const ystep = y0 < y1 ? 1 : -1;
    let y = y0;

    for (let x = x0; x <= x1; x++) {
      if (steep) {
        this.drawPixel(y, x, color);
      } else {
        this.drawPixel(x, y, color);
      }
      err -= dy;
      if (err < 0) {
        y += ystep;
        err += dx;
      }
    }
  }

  drawFastHLine(x: number, y: number, w: number, color: number): void {
    for (let i = 0; i < w; i++) this.drawPixel(x + i, y, color);
  }

  drawFastVLine(x: number, y: number, h: number, color: number): void {
    for (let i = 0; i < h; i++) this.drawPixel(x, y + i, color);
  }

  drawRect(x: number, y: number, w: number, h: number, color: number): void {
    this.drawFastHLine(x, y, w, color);
    this.drawFastHLine(x, y + h - 1, w, color);
    this.drawFastVLine(x, y, h, color);
    this.drawFastVLine(x + w - 1, y, h, color);
  }

  fillRect(x: number, y: number, w: number, h: number, color: number): void {
    for (let j = 0; j < h; j++) {
      this.drawFastHLine(x, y + j, w, color);
    }
  }

  drawCircle(x0: number, y0: number, r: number, color: number): void {
    // Midpoint circle algorithm
    let f = 1 - r;
    let ddF_x = 1;
    let ddF_y = -2 * r;
    let x = 0;
    let y = r;

    this.drawPixel(x0, y0 + r, color);
    this.drawPixel(x0, y0 - r, color);
    this.drawPixel(x0 + r, y0, color);
    this.drawPixel(x0 - r, y0, color);

    while (x < y) {
      if (f >= 0) { y--; ddF_y += 2; f += ddF_y; }
      x++; ddF_x += 2; f += ddF_x;

      this.drawPixel(x0 + x, y0 + y, color);
      this.drawPixel(x0 - x, y0 + y, color);
      this.drawPixel(x0 + x, y0 - y, color);
      this.drawPixel(x0 - x, y0 - y, color);
      this.drawPixel(x0 + y, y0 + x, color);
      this.drawPixel(x0 - y, y0 + x, color);
      this.drawPixel(x0 + y, y0 - x, color);
      this.drawPixel(x0 - y, y0 - x, color);
    }
  }

  fillCircle(x0: number, y0: number, r: number, color: number): void {
    this.drawFastVLine(x0, y0 - r, 2 * r + 1, color);
    this.fillCircleHelper(x0, y0, r, 3, 0, color);
  }

  private fillCircleHelper(
    x0: number, y0: number, r: number,
    corners: number, delta: number, color: number
  ): void {
    let f = 1 - r;
    let ddF_x = 1;
    let ddF_y = -2 * r;
    let x = 0;
    let y = r;

    while (x < y) {
      if (f >= 0) { y--; ddF_y += 2; f += ddF_y; }
      x++; ddF_x += 2; f += ddF_x;

      if (corners & 0x1) {
        this.drawFastVLine(x0 + x, y0 - y, 2 * y + 1 + delta, color);
        this.drawFastVLine(x0 + y, y0 - x, 2 * x + 1 + delta, color);
      }
      if (corners & 0x2) {
        this.drawFastVLine(x0 - x, y0 - y, 2 * y + 1 + delta, color);
        this.drawFastVLine(x0 - y, y0 - x, 2 * x + 1 + delta, color);
      }
    }
  }

  drawTriangle(
    x0: number, y0: number,
    x1: number, y1: number,
    x2: number, y2: number,
    color: number
  ): void {
    this.drawLine(x0, y0, x1, y1, color);
    this.drawLine(x1, y1, x2, y2, color);
    this.drawLine(x2, y2, x0, y0, color);
  }

  fillTriangle(
    x0: number, y0: number,
    x1: number, y1: number,
    x2: number, y2: number,
    color: number
  ): void {
    // Sort vertices by Y
    if (y0 > y1) { [x0, x1] = [x1, x0]; [y0, y1] = [y1, y0]; }
    if (y1 > y2) { [x1, x2] = [x2, x1]; [y1, y2] = [y2, y1]; }
    if (y0 > y1) { [x0, x1] = [x1, x0]; [y0, y1] = [y1, y0]; }

    if (y0 === y2) {
      let a = Math.min(x0, x1, x2);
      let b = Math.max(x0, x1, x2);
      this.drawFastHLine(a, y0, b - a + 1, color);
      return;
    }

    const dx01 = x1 - x0, dy01 = y1 - y0;
    const dx02 = x2 - x0, dy02 = y2 - y0;
    const dx12 = x2 - x1, dy12 = y2 - y1;

    let sa = 0, sb = 0;
    let last = y1 === y2 ? y1 : y1 - 1;

    for (let y = y0; y <= last; y++) {
      let a = x0 + Math.floor(sa / dy01);
      let b = x0 + Math.floor(sb / dy02);
      sa += dx01; sb += dx02;
      if (a > b) [a, b] = [b, a];
      this.drawFastHLine(a, y, b - a + 1, color);
    }

    sa = dx12 * (last + 1 - y1);
    sb = dx02 * (last + 1 - y0);
    for (let y = last + 1; y <= y2; y++) {
      let a = x1 + Math.floor(sa / dy12);
      let b = x0 + Math.floor(sb / dy02);
      sa += dx12; sb += dx02;
      if (a > b) [a, b] = [b, a];
      this.drawFastHLine(a, y, b - a + 1, color);
    }
  }

  drawRoundRect(
    x: number, y: number, w: number, h: number,
    radius: number, color: number
  ): void {
    this.drawFastHLine(x + radius, y, w - 2 * radius, color);
    this.drawFastHLine(x + radius, y + h - 1, w - 2 * radius, color);
    this.drawFastVLine(x, y + radius, h - 2 * radius, color);
    this.drawFastVLine(x + w - 1, y + radius, h - 2 * radius, color);
    // Corners via circle quadrants
    this.drawCircleQuadrant(x + radius, y + radius, radius, 1, color);
    this.drawCircleQuadrant(x + w - radius - 1, y + radius, radius, 2, color);
    this.drawCircleQuadrant(x + w - radius - 1, y + h - radius - 1, radius, 4, color);
    this.drawCircleQuadrant(x + radius, y + h - radius - 1, radius, 8, color);
  }

  fillRoundRect(
    x: number, y: number, w: number, h: number,
    radius: number, color: number
  ): void {
    this.fillRect(x + radius, y, w - 2 * radius, h, color);
    this.fillCircleHelper(x + w - radius - 1, y + radius, radius, 1, h - 2 * radius - 1, color);
    this.fillCircleHelper(x + radius, y + radius, radius, 2, h - 2 * radius - 1, color);
  }

  private drawCircleQuadrant(
    x0: number, y0: number, r: number, quadrant: number, color: number
  ): void {
    let f = 1 - r;
    let ddF_x = 1;
    let ddF_y = -2 * r;
    let x = 0;
    let y = r;

    while (x < y) {
      if (f >= 0) { y--; ddF_y += 2; f += ddF_y; }
      x++; ddF_x += 2; f += ddF_x;

      if (quadrant & 0x1) { this.drawPixel(x0 + x, y0 - y, color); this.drawPixel(x0 + y, y0 - x, color); }
      if (quadrant & 0x2) { this.drawPixel(x0 - y, y0 - x, color); this.drawPixel(x0 - x, y0 - y, color); }
      if (quadrant & 0x4) { this.drawPixel(x0 + x, y0 + y, color); this.drawPixel(x0 + y, y0 + x, color); }
      if (quadrant & 0x8) { this.drawPixel(x0 - y, y0 + x, color); this.drawPixel(x0 - x, y0 + y, color); }
    }
  }

  drawBitmap(
    x: number, y: number, bitmap: Uint8Array,
    w: number, h: number, color: number
  ): void {
    let byteIndex = 0;
    let bit = 0;

    for (let j = 0; j < h; j++) {
      for (let i = 0; i < w; i++) {
        if (i % 8 === 0 && i > 0) byteIndex++;
        if (bitmap[byteIndex] & (0x80 >> (i % 8))) {
          this.drawPixel(x + i, y + j, color);
        }
        bit++;
      }
      byteIndex++;
    }
  }

  drawChar(
    x: number, y: number, c: string,
    color: number, bg: number, size: number
  ): void {
    const charCode = c.charCodeAt(0);
    if (charCode < 0x20 || charCode > 0x7E) return;

    const fontIndex = (charCode - 0x20) * 5;

    for (let col = 0; col < 5; col++) {
      let line = FONT_5X7[fontIndex + col] ?? 0;
      for (let row = 0; row < 8; row++) {
        if (line & 0x01) {
          if (size === 1) {
            this.drawPixel(x + col, y + row, color);
          } else {
            this.fillRect(x + col * size, y + row * size, size, size, color);
          }
        } else if (bg !== color) {
          if (size === 1) {
            this.drawPixel(x + col, y + row, bg);
          } else {
            this.fillRect(x + col * size, y + row * size, size, size, bg);
          }
        }
        line >>= 1;
      }
    }

    // Column gap
    if (bg !== color) {
      if (size === 1) {
        this.drawFastVLine(x + 5, y, 8, bg);
      } else {
        this.fillRect(x + 5 * size, y, size, 8 * size, bg);
      }
    }
  }

  fillScreen(color: number): void {
    this.buffer.fill(color);
  }

  setRotation(r: number): void {
    this.rotation = r & 3;
    if (this.rotation & 1) {
      this._width = this.height;
      this._height = this.width;
    } else {
      this._width = this.width;
      this._height = this.height;
    }
  }

  setCursor(x: number, y: number): void {
    this.cursorX = x;
    this.cursorY = y;
  }

  setTextColor(fg: number, bg?: number): void {
    this.textColor = fg;
    this.textBg = bg ?? fg;
  }

  setTextSize(s: number): void {
    this.textSize = Math.max(1, s);
  }

  setTextWrap(wrap: boolean): void {
    this.wrap = wrap;
  }

  print(text: string): void {
    for (const c of text) {
      if (c === '\n') {
        this.cursorX = 0;
        this.cursorY += this.textSize * 8;
        continue;
      }
      if (c === '\r') {
        this.cursorX = 0;
        continue;
      }

      if (this.wrap && (this.cursorX + this.textSize * 6 > this._width)) {
        this.cursorX = 0;
        this.cursorY += this.textSize * 8;
      }

      this.drawChar(
        this.cursorX, this.cursorY, c,
        this.textColor, this.textBg, this.textSize
      );
      this.cursorX += this.textSize * 6; // 5 pixels + 1 gap
    }
  }

  println(text: string): void {
    this.print(text + '\n');
  }

  color565(r: number, g: number, b: number): number {
    return rgb888ToRgb565(r, g, b);
  }
}
```

### Pattern 4: Color Display Controller Emulator (SPI Command Protocol)

Color TFT and OLED displays (ILI9341, ST7735, SSD1331) use an SPI-like command protocol with a Data/Command (DC) pin. Here's a virtual color display controller that accepts the same command bytes:

```typescript
// Common display controller commands shared across SSD1331/ST7735/ILI9341
enum DisplayCmd {
  // System
  NOP          = 0x00,
  SWRESET      = 0x01,  // Software reset
  SLPIN        = 0x10,  // Sleep in
  SLPOUT       = 0x11,  // Sleep out
  NORON        = 0x13,  // Normal display mode on
  INVOFF       = 0x20,  // Display inversion off
  INVON        = 0x21,  // Display inversion on
  DISPOFF      = 0x28,  // Display off
  DISPON       = 0x29,  // Display on

  // Addressing
  CASET        = 0x2A,  // Column address set (x start/end)
  RASET        = 0x2B,  // Row address set (y start/end)
  RAMWR        = 0x2C,  // Memory write (pixel data follows)
  RAMRD        = 0x2E,  // Memory read

  // Display control
  MADCTL       = 0x36,  // Memory Access Data Control (rotation/mirror)
  COLMOD       = 0x3A,  // Color mode (12/16/18 bit)

  // Scrolling
  VSCRDEF      = 0x33,  // Vertical scrolling definition
  VSCRSADD     = 0x37,  // Vertical scrolling start address
}

// MADCTL bit flags (same across ST7735/ILI9341/SSD1331)
const MADCTL_MY  = 0x80; // Row address order (mirror Y)
const MADCTL_MX  = 0x40; // Column address order (mirror X)
const MADCTL_MV  = 0x20; // Row/column exchange (rotate 90)
const MADCTL_ML  = 0x10; // Vertical refresh order
const MADCTL_BGR = 0x08; // BGR color order (vs RGB)
const MADCTL_MH  = 0x04; // Horizontal refresh order

interface DisplayControllerConfig {
  width: number;
  height: number;
  colorDepth: 12 | 16 | 18;  // Bits per pixel
  name: string;               // 'SSD1331' | 'ST7735' | 'ILI9341'
}

const DISPLAY_CONFIGS: Record<string, DisplayControllerConfig> = {
  SSD1331: { width: 96, height: 64, colorDepth: 16, name: 'SSD1331' },
  ST7735:  { width: 128, height: 160, colorDepth: 16, name: 'ST7735' },
  ILI9341: { width: 240, height: 320, colorDepth: 16, name: 'ILI9341' },
  SSD1306: { width: 128, height: 64, colorDepth: 1 as any, name: 'SSD1306' },
};

class VirtualDisplayController {
  private framebuffer: Uint16Array;
  private config: DisplayControllerConfig;

  // Address window for pixel writes
  private colStart = 0;
  private colEnd: number;
  private rowStart = 0;
  private rowEnd: number;
  private writeCol = 0;
  private writeRow = 0;

  // State
  private displayOn = true;
  private inverted = false;
  private sleeping = false;
  private madctl = 0x00;
  private colorMode = 16;

  // Command parsing state
  private currentCommand: number | null = null;
  private paramBuffer: number[] = [];
  private expectedParams = 0;

  // Scroll
  private scrollTopFixed = 0;
  private scrollArea = 0;
  private scrollOffset = 0;

  private listeners: ((fb: Uint16Array) => void)[] = [];

  constructor(configName: string = 'ST7735') {
    this.config = DISPLAY_CONFIGS[configName] ?? DISPLAY_CONFIGS.ST7735;
    this.framebuffer = new Uint16Array(this.config.width * this.config.height);
    this.colEnd = this.config.width - 1;
    this.rowEnd = this.config.height - 1;
  }

  onFrame(listener: (fb: Uint16Array) => void): void {
    this.listeners.push(listener);
  }

  // DC=0: Command byte
  writeCommand(cmd: number): void {
    this.currentCommand = cmd;
    this.paramBuffer = [];

    switch (cmd) {
      case DisplayCmd.SWRESET:
        this.reset();
        break;
      case DisplayCmd.SLPOUT:
        this.sleeping = false;
        break;
      case DisplayCmd.SLPIN:
        this.sleeping = true;
        break;
      case DisplayCmd.DISPON:
        this.displayOn = true;
        this.emitFrame();
        break;
      case DisplayCmd.DISPOFF:
        this.displayOn = false;
        break;
      case DisplayCmd.INVON:
        this.inverted = true;
        this.emitFrame();
        break;
      case DisplayCmd.INVOFF:
        this.inverted = false;
        this.emitFrame();
        break;
      case DisplayCmd.CASET:
        this.expectedParams = 4;
        break;
      case DisplayCmd.RASET:
        this.expectedParams = 4;
        break;
      case DisplayCmd.MADCTL:
        this.expectedParams = 1;
        break;
      case DisplayCmd.COLMOD:
        this.expectedParams = 1;
        break;
      case DisplayCmd.RAMWR:
        this.writeCol = this.colStart;
        this.writeRow = this.rowStart;
        this.expectedParams = Infinity;
        break;
      case DisplayCmd.VSCRDEF:
        this.expectedParams = 6;
        break;
      case DisplayCmd.VSCRSADD:
        this.expectedParams = 2;
        break;
    }
  }

  // DC=1: Data byte
  writeData(byte: number): void {
    this.paramBuffer.push(byte);

    if (this.currentCommand === DisplayCmd.RAMWR) {
      this.handlePixelData();
      return;
    }

    if (this.paramBuffer.length >= this.expectedParams) {
      this.executeCommand();
    }
  }

  // Bulk data write (for efficiency)
  writeDataBulk(data: Uint8Array | number[]): void {
    for (const byte of data) {
      this.writeData(byte);
    }
  }

  private handlePixelData(): void {
    if (this.paramBuffer.length < 2) return;

    // RGB565: 2 bytes per pixel, big-endian
    const hi = this.paramBuffer[this.paramBuffer.length - 2];
    const lo = this.paramBuffer[this.paramBuffer.length - 1];

    if (this.paramBuffer.length % 2 !== 0) return;

    const color = (hi << 8) | lo;
    const idx = this.writeRow * this.config.width + this.writeCol;

    if (idx < this.framebuffer.length) {
      this.framebuffer[idx] = this.inverted ? ~color & 0xFFFF : color;
    }

    // Advance write position within window
    this.writeCol++;
    if (this.writeCol > this.colEnd) {
      this.writeCol = this.colStart;
      this.writeRow++;
      if (this.writeRow > this.rowEnd) {
        this.writeRow = this.rowStart;
        this.emitFrame();
      }
    }

    // Keep only the last incomplete pixel
    if (this.paramBuffer.length >= 2) {
      this.paramBuffer = [];
    }
  }

  private executeCommand(): void {
    const p = this.paramBuffer;

    switch (this.currentCommand) {
      case DisplayCmd.CASET:
        this.colStart = (p[0] << 8) | p[1];
        this.colEnd = (p[2] << 8) | p[3];
        break;

      case DisplayCmd.RASET:
        this.rowStart = (p[0] << 8) | p[1];
        this.rowEnd = (p[2] << 8) | p[3];
        break;

      case DisplayCmd.MADCTL:
        this.madctl = p[0];
        break;

      case DisplayCmd.COLMOD:
        this.colorMode = p[0] & 0x07;
        break;

      case DisplayCmd.VSCRDEF:
        this.scrollTopFixed = (p[0] << 8) | p[1];
        this.scrollArea = (p[2] << 8) | p[3];
        break;

      case DisplayCmd.VSCRSADD:
        this.scrollOffset = (p[0] << 8) | p[1];
        this.emitFrame();
        break;
    }

    this.currentCommand = null;
    this.paramBuffer = [];
  }

  private reset(): void {
    this.framebuffer.fill(0x0000);
    this.displayOn = true;
    this.inverted = false;
    this.sleeping = false;
    this.madctl = 0x00;
    this.colStart = 0;
    this.colEnd = this.config.width - 1;
    this.rowStart = 0;
    this.rowEnd = this.config.height - 1;
    this.writeCol = 0;
    this.writeRow = 0;
    this.scrollOffset = 0;
  }

  private emitFrame(): void {
    for (const listener of this.listeners) {
      listener(this.framebuffer);
    }
  }

  getFramebuffer(): Uint16Array {
    return this.framebuffer;
  }

  getConfig(): DisplayControllerConfig {
    return this.config;
  }
}
```

### Pattern 5: WebSocket Protocol Adapter

Accept display commands over WebSocket so remote processes can drive your virtual display. This bridges the gap between embedded LCD libraries and a macOS Canvas renderer.

```typescript
interface DisplayProtocolMessage {
  // Instruction messages (RS=0)
  type: 'instruction' | 'data' | 'data-bulk' | 'text' | 'clear'
      | 'cursor' | 'gfx' | 'theme' | 'spi-cmd' | 'spi-data';

  // For instruction/data: the byte value
  byte?: number;

  // For data-bulk: array of bytes (efficient pixel writes)
  bytes?: number[];

  // For text: string to display
  text?: string;
  row?: number;
  col?: number;

  // For cursor: position
  x?: number;
  y?: number;

  // For gfx: drawing primitives
  op?: 'drawPixel' | 'drawLine' | 'drawRect' | 'fillRect'
     | 'drawCircle' | 'fillCircle' | 'drawBitmap' | 'fillScreen'
     | 'print' | 'println' | 'drawChar';
  args?: number[];
  color?: number;

  // For theme: change display appearance
  theme?: string;
}

// LCDproc-compatible text protocol (ASCII over TCP/WebSocket)
// Based on the LCDproc client-server protocol on port 13666
class LCDProcProtocolAdapter {
  private lcd: VirtualHD44780;
  private clientName = 'unknown';

  constructor(lcd: VirtualHD44780) {
    this.lcd = lcd;
  }

  // Parse and execute an LCDproc protocol line
  handleLine(line: string): string {
    const parts = line.trim().split(/\s+/);
    const cmd = parts[0];

    switch (cmd) {
      case 'hello':
        return `connect LCDproc 0.5.9 protocol 0.3 lcd wid ${this.lcd.cols} hgt ${this.lcd.rows} cellwid 5 cellhgt 8`;

      case 'client_set':
        if (parts[1] === '-name') {
          this.clientName = parts[2] ?? 'unknown';
        }
        return 'success';

      case 'screen_add':
        return 'success';

      case 'screen_set':
        return 'success';

      case 'widget_add': {
        const _screenId = parts[1];
        const _widgetId = parts[2];
        const _widgetType = parts[3]; // string, hbar, vbar, title, scroller
        return 'success';
      }

      case 'widget_set': {
        const _screenId = parts[1];
        const widgetId = parts[2];
        // widget_set screen widget x y "text"
        if (parts.length >= 5) {
          const x = parseInt(parts[3]) - 1; // LCDproc is 1-indexed
          const y = parseInt(parts[4]) - 1;
          // Extract quoted text
          const textMatch = line.match(/"([^"]*)"/);
          if (textMatch) {
            this.lcd.setCursor(y, x);
            this.lcd.printString(textMatch[1]);
          }
        }
        return 'success';
      }

      case 'backlight':
        // on/off/toggle/blink
        if (parts[1] === 'on') {
          this.lcd.writeInstruction(0x0C); // Display on
        } else if (parts[1] === 'off') {
          this.lcd.writeInstruction(0x08); // Display off
        }
        return 'success';

      default:
        return `huh? unknown command: ${cmd}`;
    }
  }
}

// WebSocket server that accepts display protocol messages
function createDisplayServer(
  lcd: VirtualHD44780,
  gfx: PixelBuffer,
  colorDisplay: VirtualDisplayController,
  port = 13666
): void {
  // In a real implementation, use 'ws' package or Deno.serve
  // This shows the message handling logic

  const lcdproc = new LCDProcProtocolAdapter(lcd);

  function handleMessage(raw: string | ArrayBuffer): string | void {
    // Text messages: try LCDproc protocol first, then JSON
    if (typeof raw === 'string') {
      // Try JSON parse
      try {
        const msg: DisplayProtocolMessage = JSON.parse(raw);
        return handleJsonMessage(msg);
      } catch {
        // Fall back to LCDproc text protocol
        return lcdproc.handleLine(raw);
      }
    }

    // Binary messages: treat as raw SPI data stream
    if (raw instanceof ArrayBuffer) {
      const data = new Uint8Array(raw);
      for (const byte of data) {
        colorDisplay.writeData(byte);
      }
    }
  }

  function handleJsonMessage(msg: DisplayProtocolMessage): string {
    switch (msg.type) {
      case 'instruction':
        lcd.writeInstruction(msg.byte!);
        return 'ok';

      case 'data':
        lcd.writeData(msg.byte!);
        return 'ok';

      case 'data-bulk':
        for (const byte of msg.bytes ?? []) {
          lcd.writeData(byte);
        }
        return 'ok';

      case 'text':
        if (msg.row !== undefined && msg.col !== undefined) {
          lcd.setCursor(msg.row, msg.col);
        }
        lcd.printString(msg.text ?? '');
        return 'ok';

      case 'clear':
        lcd.writeInstruction(0x01);
        return 'ok';

      case 'cursor':
        lcd.setCursor(msg.row ?? 0, msg.col ?? 0);
        return 'ok';

      case 'gfx':
        return handleGfxMessage(msg);

      case 'spi-cmd':
        colorDisplay.writeCommand(msg.byte!);
        return 'ok';

      case 'spi-data':
        if (msg.bytes) {
          colorDisplay.writeDataBulk(msg.bytes);
        } else if (msg.byte !== undefined) {
          colorDisplay.writeData(msg.byte);
        }
        return 'ok';

      default:
        return 'error: unknown message type';
    }
  }

  function handleGfxMessage(msg: DisplayProtocolMessage): string {
    const a = msg.args ?? [];
    const c = msg.color ?? 0xFFFF;

    switch (msg.op) {
      case 'drawPixel':   gfx.drawPixel(a[0], a[1], c); break;
      case 'drawLine':    gfx.drawLine(a[0], a[1], a[2], a[3], c); break;
      case 'drawRect':    gfx.drawRect(a[0], a[1], a[2], a[3], c); break;
      case 'fillRect':    gfx.fillRect(a[0], a[1], a[2], a[3], c); break;
      case 'drawCircle':  gfx.drawCircle(a[0], a[1], a[2], c); break;
      case 'fillCircle':  gfx.fillCircle(a[0], a[1], a[2], c); break;
      case 'fillScreen':  gfx.fillScreen(c); break;
      case 'print':       gfx.print(msg.text ?? ''); break;
      case 'println':     gfx.println(msg.text ?? ''); break;
      case 'drawChar':    gfx.drawChar(a[0], a[1], msg.text?.[0] ?? ' ', c, a[2] ?? 0, a[3] ?? 1); break;
      default: return 'error: unknown gfx op';
    }
    return 'ok';
  }
}
```

### Pattern 6: HD44780-to-Canvas Bridge

Connect the virtual HD44780 to the Canvas renderer. The HD44780 stores character codes in DDRAM; this bridge looks up the 5x7 font and renders each character as dots.

```typescript
class HD44780CanvasBridge {
  private lcd: VirtualHD44780;
  private renderer: DotMatrixRenderer;
  private canvas: HTMLCanvasElement;

  // Character cell dimensions
  private charWidth = 5;   // pixels per character width
  private charHeight = 8;  // pixels per character height (7 + 1 gap)
  private charGapX = 1;    // gap between characters
  private charGapY = 1;    // gap between rows

  // Total pixel grid
  private gridWidth: number;
  private gridHeight: number;
  private pixels: Uint8Array;

  // Cursor blink state
  private blinkVisible = true;
  private blinkTimer: ReturnType<typeof setInterval> | null = null;

  constructor(
    lcd: VirtualHD44780,
    canvas: HTMLCanvasElement,
    theme: DotMatrixTheme = THEMES.classicGreen
  ) {
    this.lcd = lcd;
    this.canvas = canvas;

    this.gridWidth = lcd.cols * (this.charWidth + this.charGapX) - this.charGapX;
    this.gridHeight = lcd.rows * (this.charHeight + this.charGapY) - this.charGapY;
    this.pixels = new Uint8Array(this.gridWidth * this.gridHeight);

    this.renderer = new DotMatrixRenderer(canvas, this.gridWidth, this.gridHeight, theme);

    // Listen for LCD state changes
    lcd.onEvent(() => this.render());

    // Start cursor blink timer
    this.blinkTimer = setInterval(() => {
      this.blinkVisible = !this.blinkVisible;
      if (lcd.cursorBlink || lcd.cursorOn) {
        this.render();
      }
    }, 530); // HD44780 blink rate is approximately 1.9 Hz
  }

  render(): void {
    this.pixels.fill(0);

    if (!this.lcd.displayOn) {
      this.renderer.renderMono(this.pixels);
      return;
    }

    const rowOffsets = this.lcd.rows === 4
      ? [0x00, 0x40, this.lcd.cols, 0x40 + this.lcd.cols]
      : [0x00, 0x40];

    for (let row = 0; row < this.lcd.rows; row++) {
      for (let col = 0; col < this.lcd.cols; col++) {
        const addr = (rowOffsets[row] + col + this.lcd.shiftOffset) & 0x7F;
        const charCode = this.lcd.ddram[addr];

        this.renderCharacter(charCode, row, col);
      }
    }

    // Render cursor
    if (this.lcd.cursorOn || this.lcd.cursorBlink) {
      const pos = this.lcd.getCursorPosition();
      this.renderCursor(pos.row, pos.col);
    }

    this.renderer.renderMono(this.pixels);
  }

  private renderCharacter(charCode: number, row: number, col: number): void {
    const startX = col * (this.charWidth + this.charGapX);
    const startY = row * (this.charHeight + this.charGapY);

    if (charCode < 8) {
      // CGRAM custom character
      const bitmap = this.lcd.getCustomCharacter(charCode);
      for (let py = 0; py < 8; py++) {
        for (let px = 0; px < 5; px++) {
          if (bitmap[py] & (0x10 >> px)) {
            const idx = (startY + py) * this.gridWidth + (startX + px);
            if (idx < this.pixels.length) {
              this.pixels[idx] = 1;
            }
          }
        }
      }
    } else if (charCode >= 0x20 && charCode <= 0x7E) {
      // Standard ASCII from 5x7 font
      const fontIndex = (charCode - 0x20) * 5;
      for (let px = 0; px < 5; px++) {
        const column = FONT_5X7[fontIndex + px] ?? 0;
        for (let py = 0; py < 7; py++) {
          if (column & (1 << py)) {
            const idx = (startY + py) * this.gridWidth + (startX + px);
            if (idx < this.pixels.length) {
              this.pixels[idx] = 1;
            }
          }
        }
      }
    }
  }

  private renderCursor(row: number, col: number): void {
    const startX = col * (this.charWidth + this.charGapX);
    const startY = row * (this.charHeight + this.charGapY);

    if (this.lcd.cursorBlink && !this.blinkVisible) {
      return; // Blink off phase
    }

    if (this.lcd.cursorBlink) {
      // Block cursor (fill entire character cell)
      for (let py = 0; py < this.charHeight; py++) {
        for (let px = 0; px < this.charWidth; px++) {
          const idx = (startY + py) * this.gridWidth + (startX + px);
          if (idx < this.pixels.length) {
            this.pixels[idx] = 1;
          }
        }
      }
    } else if (this.lcd.cursorOn) {
      // Underline cursor (bottom row only)
      const py = this.charHeight - 1;
      for (let px = 0; px < this.charWidth; px++) {
        const idx = (startY + py) * this.gridWidth + (startX + px);
        if (idx < this.pixels.length) {
          this.pixels[idx] = 1;
        }
      }
    }
  }

  destroy(): void {
    if (this.blinkTimer) {
      clearInterval(this.blinkTimer);
      this.blinkTimer = null;
    }
  }
}
```

### Pattern 7: Scrolling Text Engine

Horizontal and vertical scrolling for LED matrix displays. This is one of the most common use cases — scrolling text that's wider than the display.

```typescript
interface ScrollOptions {
  direction: 'left' | 'right' | 'up' | 'down';
  speed: number;        // Pixels per frame
  gap: number;          // Pixels of gap before text repeats
  loop: boolean;        // Whether to loop continuously
  color: number;        // RGB565 color
  bgColor?: number;     // Background color (default: black)
  font?: 'small' | 'medium' | 'large'; // Text size multiplier
}

class ScrollingTextEngine {
  private gfx: PixelBuffer;
  private animFrame: number | null = null;

  // Active scroll regions
  private scrollers: Map<string, ScrollState> = new Map();

  constructor(gfx: PixelBuffer) {
    this.gfx = gfx;
  }

  addScroller(
    id: string,
    text: string,
    y: number,
    height: number,
    options: Partial<ScrollOptions> = {}
  ): void {
    const opts: ScrollOptions = {
      direction: 'left',
      speed: 1,
      gap: 20,
      loop: true,
      color: 0xFFFF,
      bgColor: 0x0000,
      font: 'small',
      ...options,
    };

    const sizeMultiplier = opts.font === 'large' ? 3 : opts.font === 'medium' ? 2 : 1;
    const charWidth = 6 * sizeMultiplier; // 5px + 1px gap
    const textWidth = text.length * charWidth;

    this.scrollers.set(id, {
      text,
      y,
      height,
      options: opts,
      offset: 0,
      textWidth,
      sizeMultiplier,
    });

    if (!this.animFrame) {
      this.startAnimation();
    }
  }

  removeScroller(id: string): void {
    this.scrollers.delete(id);
    if (this.scrollers.size === 0 && this.animFrame) {
      cancelAnimationFrame(this.animFrame);
      this.animFrame = null;
    }
  }

  updateText(id: string, text: string): void {
    const state = this.scrollers.get(id);
    if (state) {
      state.text = text;
      state.textWidth = text.length * 6 * state.sizeMultiplier;
    }
  }

  private startAnimation(): void {
    const tick = () => {
      for (const [, state] of this.scrollers) {
        this.renderScroller(state);
        state.offset += state.options.speed;

        const totalWidth = state.textWidth + state.options.gap;
        if (state.offset >= totalWidth) {
          if (state.options.loop) {
            state.offset = 0;
          } else {
            state.offset = totalWidth;
          }
        }
      }
      this.animFrame = requestAnimationFrame(tick);
    };
    this.animFrame = requestAnimationFrame(tick);
  }

  private renderScroller(state: ScrollState): void {
    const { text, y, height, options, offset, sizeMultiplier } = state;
    const bg = options.bgColor ?? 0x0000;

    // Clear scroll region
    this.gfx.fillRect(0, y, this.gfx.width, height, bg);

    // Save and set text properties
    this.gfx.setTextSize(sizeMultiplier);
    this.gfx.setTextColor(options.color, bg);
    this.gfx.setTextWrap(false);

    const textY = y + Math.floor((height - 8 * sizeMultiplier) / 2);

    if (options.direction === 'left') {
      // Draw text at offset position
      this.gfx.setCursor(-offset, textY);
      this.gfx.print(text);

      // Draw repeated copy for seamless loop
      if (options.loop) {
        const repeatX = state.textWidth + options.gap - offset;
        this.gfx.setCursor(repeatX, textY);
        this.gfx.print(text);
      }
    } else if (options.direction === 'right') {
      this.gfx.setCursor(offset - state.textWidth, textY);
      this.gfx.print(text);
    }
  }

  stop(): void {
    if (this.animFrame) {
      cancelAnimationFrame(this.animFrame);
      this.animFrame = null;
    }
  }
}

interface ScrollState {
  text: string;
  y: number;
  height: number;
  options: ScrollOptions;
  offset: number;
  textWidth: number;
  sizeMultiplier: number;
}
```

---

## Small Examples

### Example 1: Custom Character Definitions (HD44780 CGRAM)

```typescript
// The HD44780 supports 8 custom characters (indices 0-7)
// Each character is 5 pixels wide and 8 rows tall
// Only the lower 5 bits of each row are used

const CUSTOM_CHARS = {
  heart: [
    0b00000,
    0b01010,
    0b11111,
    0b11111,
    0b11111,
    0b01110,
    0b00100,
    0b00000,
  ],
  smiley: [
    0b00000,
    0b01010,
    0b00000,
    0b00000,
    0b10001,
    0b01110,
    0b00000,
    0b00000,
  ],
  battery_full: [
    0b01110,
    0b11111,
    0b11111,
    0b11111,
    0b11111,
    0b11111,
    0b11111,
    0b11111,
  ],
  battery_empty: [
    0b01110,
    0b10001,
    0b10001,
    0b10001,
    0b10001,
    0b10001,
    0b10001,
    0b11111,
  ],
  arrow_right: [
    0b01000,
    0b01100,
    0b01110,
    0b01111,
    0b01110,
    0b01100,
    0b01000,
    0b00000,
  ],
  thermometer: [
    0b00100,
    0b01010,
    0b01010,
    0b01010,
    0b01010,
    0b10001,
    0b10001,
    0b01110,
  ],
  wifi: [
    0b00000,
    0b01110,
    0b10001,
    0b00100,
    0b01010,
    0b00000,
    0b00100,
    0b00000,
  ],
  lock: [
    0b01110,
    0b10001,
    0b10001,
    0b11111,
    0b11011,
    0b11011,
    0b11111,
    0b00000,
  ],
};

// Load all custom characters
const lcd = new VirtualHD44780(20, 4);
Object.values(CUSTOM_CHARS).forEach((bitmap, i) => {
  if (i < 8) lcd.defineCharacter(i, bitmap);
});

// Display: "Status: [heart] OK [battery] [wifi]"
lcd.setCursor(0, 0);
lcd.printString('Status: ');
lcd.writeData(0); // heart
lcd.printString(' OK ');
lcd.writeData(2); // battery_full
lcd.writeData(6); // wifi
```

### Example 2: Mini Bar Chart on RGB Display

```typescript
function drawBarChart(
  gfx: PixelBuffer,
  x: number, y: number,
  width: number, height: number,
  values: number[],        // 0.0 to 1.0
  colors: number[],        // RGB565 color per bar
  bgColor: number = 0x0000,
  barGap: number = 1,
): void {
  const barWidth = Math.floor(
    (width - barGap * (values.length - 1)) / values.length
  );

  gfx.fillRect(x, y, width, height, bgColor);

  for (let i = 0; i < values.length; i++) {
    const barHeight = Math.round(values[i] * height);
    const barX = x + i * (barWidth + barGap);
    const barY = y + height - barHeight;
    const color = colors[i % colors.length];

    gfx.fillRect(barX, barY, barWidth, barHeight, color);
  }
}

// Usage: CPU usage per core on a 64x32 matrix
const display = new PixelBuffer(64, 32);
const cpuValues = [0.85, 0.42, 0.67, 0.93, 0.31, 0.55, 0.78, 0.22];
const cpuColors = cpuValues.map(v =>
  v > 0.8 ? 0xF800 :  // Red for >80%
  v > 0.5 ? 0xFFE0 :  // Yellow for >50%
            0x07E0     // Green for <50%
);

drawBarChart(display, 0, 8, 64, 24, cpuValues, cpuColors);
display.setTextSize(1);
display.setTextColor(0xFFFF);
display.setCursor(0, 0);
display.print('CPU');
```

### Example 3: Status Icon Sprites

```typescript
// 8x8 sprites stored as bitmaps (1 bit per pixel, row-major)
const SPRITES_8X8: Record<string, Uint8Array> = {
  check: new Uint8Array([
    0b00000000,
    0b00000001,
    0b00000010,
    0b10000100,
    0b01001000,
    0b00110000,
    0b00000000,
    0b00000000,
  ]),
  cross: new Uint8Array([
    0b00000000,
    0b01000010,
    0b00100100,
    0b00011000,
    0b00011000,
    0b00100100,
    0b01000010,
    0b00000000,
  ]),
  warning: new Uint8Array([
    0b00011000,
    0b00011000,
    0b00100100,
    0b00100100,
    0b01000010,
    0b01011010,
    0b10000001,
    0b11111111,
  ]),
  clock: new Uint8Array([
    0b00111100,
    0b01000010,
    0b10010001,
    0b10010001,
    0b10001111,
    0b10000001,
    0b01000010,
    0b00111100,
  ]),
};

function drawSprite(
  gfx: PixelBuffer,
  sprite: Uint8Array,
  x: number, y: number,
  color: number,
  scale: number = 1
): void {
  for (let row = 0; row < 8; row++) {
    for (let col = 0; col < 8; col++) {
      if (sprite[row] & (0x80 >> col)) {
        if (scale === 1) {
          gfx.drawPixel(x + col, y + row, color);
        } else {
          gfx.fillRect(x + col * scale, y + row * scale, scale, scale, color);
        }
      }
    }
  }
}

// Draw status row: [check] OK  [warning] 3
const display = new PixelBuffer(64, 32);
drawSprite(display, SPRITES_8X8.check, 0, 0, 0x07E0);  // Green check
display.setCursor(10, 0);
display.setTextColor(0x07E0);
display.print('OK');

drawSprite(display, SPRITES_8X8.warning, 32, 0, 0xFFE0); // Yellow warning
display.setCursor(42, 0);
display.setTextColor(0xFFE0);
display.print('3');
```

### Example 4: Color Temperature Display

```typescript
function drawTemperatureGauge(
  gfx: PixelBuffer,
  temp: number,          // Current temperature
  min: number,           // Min of range
  max: number,           // Max of range
  x: number, y: number,
  width: number, height: number,
): void {
  const normalized = Math.max(0, Math.min(1, (temp - min) / (max - min)));

  // Color gradient: blue (cold) -> green (normal) -> red (hot)
  let color: number;
  if (normalized < 0.5) {
    // Blue to green
    const t = normalized * 2;
    const r = 0;
    const g = Math.round(t * 255);
    const b = Math.round((1 - t) * 255);
    color = rgb888ToRgb565(r, g, b);
  } else {
    // Green to red
    const t = (normalized - 0.5) * 2;
    const r = Math.round(t * 255);
    const g = Math.round((1 - t) * 255);
    const b = 0;
    color = rgb888ToRgb565(r, g, b);
  }

  // Draw gauge background
  gfx.drawRect(x, y, width, height, 0x4208); // Dark gray border
  const barWidth = Math.round(normalized * (width - 2));
  gfx.fillRect(x + 1, y + 1, barWidth, height - 2, color);

  // Draw temperature text
  const text = `${Math.round(temp)}C`;
  gfx.setTextColor(0xFFFF);
  gfx.setTextSize(1);
  gfx.setCursor(x + 2, y + Math.floor((height - 8) / 2));
  gfx.print(text);
}

// Usage
const display = new PixelBuffer(128, 64);
display.setTextColor(0xFFFF);
display.setCursor(0, 0);
display.print('System Temp');
drawTemperatureGauge(display, 67, 20, 100, 0, 12, 128, 14);

display.setCursor(0, 30);
display.print('GPU Temp');
drawTemperatureGauge(display, 82, 20, 100, 0, 42, 128, 14);
```

### Example 5: WLED-Compatible JSON Controller

```typescript
// Accept WLED JSON API messages and apply to a pixel buffer
// Based on the WLED JSON API specification at kno.wled.ge

interface WLEDState {
  on: boolean;
  bri: number;        // 0-255 brightness
  transition: number; // Transition time in 100ms units
  seg: WLEDSegment[];
}

interface WLEDSegment {
  id: number;
  start: number;
  stop: number;
  col: [number, number, number][]; // Up to 3 colors, each [R, G, B]
  fx: number;      // Effect ID
  sx: number;      // Effect speed
  ix: number;      // Effect intensity
  pal: number;     // Palette ID
}

class WLEDAdapter {
  private state: WLEDState;
  private pixels: Uint8Array; // RGB888, 3 bytes per pixel
  private numPixels: number;

  constructor(numPixels: number) {
    this.numPixels = numPixels;
    this.pixels = new Uint8Array(numPixels * 3);
    this.state = {
      on: true,
      bri: 128,
      transition: 7,
      seg: [{
        id: 0,
        start: 0,
        stop: numPixels,
        col: [[255, 160, 0], [0, 0, 0], [0, 0, 0]],
        fx: 0,
        sx: 128,
        ix: 128,
        pal: 0,
      }],
    };
  }

  // Apply a partial state update (WLED JSON API format)
  applyState(update: Partial<WLEDState>): void {
    if (update.on !== undefined) this.state.on = update.on;
    if (update.bri !== undefined) this.state.bri = update.bri;
    if (update.seg) {
      for (const segUpdate of update.seg) {
        const existing = this.state.seg.find(s => s.id === segUpdate.id);
        if (existing) {
          Object.assign(existing, segUpdate);
        }
      }
    }
    this.render();
  }

  // Render current state to pixel buffer
  private render(): void {
    if (!this.state.on) {
      this.pixels.fill(0);
      return;
    }

    const brightness = this.state.bri / 255;

    for (const seg of this.state.seg) {
      const [r, g, b] = seg.col[0];
      for (let i = seg.start; i < seg.stop && i < this.numPixels; i++) {
        this.pixels[i * 3] = Math.round(r * brightness);
        this.pixels[i * 3 + 1] = Math.round(g * brightness);
        this.pixels[i * 3 + 2] = Math.round(b * brightness);
      }
    }
  }

  getPixels(): Uint8Array {
    return this.pixels;
  }

  // Return current state (for WebSocket GET)
  getState(): WLEDState {
    return { ...this.state };
  }
}
```

### Example 6: Animated Progress Spinner

```typescript
function drawSpinner(
  gfx: PixelBuffer,
  cx: number, cy: number,
  radius: number,
  frame: number,        // 0-7 for 8-frame animation
  color: number,
  bgColor: number = 0x0000,
): void {
  // 8-segment spinner, each segment is an arc
  const segments = 8;
  const segmentAngle = (2 * Math.PI) / segments;

  for (let i = 0; i < segments; i++) {
    const angle = i * segmentAngle - Math.PI / 2;
    const intensity = ((i + frame) % segments) / segments;
    const x = Math.round(cx + Math.cos(angle) * radius);
    const y = Math.round(cy + Math.sin(angle) * radius);

    // Fade from dim to bright based on position relative to frame
    if (intensity > 0.3) {
      gfx.drawPixel(x, y, color);
    } else {
      gfx.drawPixel(x, y, bgColor);
    }
  }
}

// Animate on a 32x32 display
const display = new PixelBuffer(32, 32);
let frame = 0;
setInterval(() => {
  display.fillScreen(0x0000);
  drawSpinner(display, 16, 16, 8, frame, 0x07FF);  // Cyan spinner

  display.setTextColor(0xFFFF);
  display.setTextSize(1);
  display.setCursor(4, 26);
  display.print('LOAD');

  frame = (frame + 1) % 8;
}, 100);
```

### Example 7: Stock Ticker with Red/Green

```typescript
interface TickerItem {
  symbol: string;
  price: number;
  change: number; // Percentage, positive or negative
}

function drawStockTicker(
  gfx: PixelBuffer,
  items: TickerItem[],
  y: number,
  height: number,
): void {
  gfx.fillRect(0, y, gfx.width, height, 0x0000);

  let x = 2;
  gfx.setTextSize(1);

  for (const item of items) {
    const isUp = item.change >= 0;
    const symbolColor = 0xFFFF; // White
    const priceColor = isUp ? 0x07E0 : 0xF800; // Green or Red
    const arrow = isUp ? '\x1E' : '\x1F'; // Up/down arrows
    const changeStr = `${isUp ? '+' : ''}${item.change.toFixed(1)}%`;

    // Symbol
    gfx.setTextColor(symbolColor);
    gfx.setCursor(x, y + 1);
    gfx.print(item.symbol);
    x += item.symbol.length * 6 + 2;

    // Price + change
    gfx.setTextColor(priceColor);
    gfx.setCursor(x, y + 1);
    gfx.print(`${item.price.toFixed(0)} ${changeStr}`);
    x += (changeStr.length + item.price.toFixed(0).length + 1) * 6 + 8;
  }
}

// Usage on 128x32 matrix
const display = new PixelBuffer(128, 32);
drawStockTicker(display, [
  { symbol: 'AAPL', price: 178.2, change: 1.3 },
  { symbol: 'GOOGL', price: 142.5, change: -0.8 },
], 0, 10);
```

### Example 8: Multi-Zone Dashboard Layout

```typescript
interface DashboardZone {
  x: number;
  y: number;
  width: number;
  height: number;
  type: 'text' | 'bar' | 'icon' | 'sparkline';
  label: string;
  value: string | number | number[];
  color: number;
}

function renderDashboard(gfx: PixelBuffer, zones: DashboardZone[]): void {
  gfx.fillScreen(0x0000);

  for (const zone of zones) {
    // Draw zone border
    gfx.drawRect(zone.x, zone.y, zone.width, zone.height, 0x2104);

    switch (zone.type) {
      case 'text': {
        gfx.setTextColor(0x8410); // Gray label
        gfx.setTextSize(1);
        gfx.setCursor(zone.x + 2, zone.y + 1);
        gfx.print(zone.label);

        gfx.setTextColor(zone.color);
        gfx.setCursor(zone.x + 2, zone.y + 10);
        gfx.print(String(zone.value));
        break;
      }
      case 'bar': {
        const val = typeof zone.value === 'number' ? zone.value : 0;
        const barW = Math.round(val * (zone.width - 4));
        gfx.fillRect(zone.x + 2, zone.y + 2, barW, zone.height - 4, zone.color);

        gfx.setTextColor(0xFFFF);
        gfx.setTextSize(1);
        gfx.setCursor(zone.x + 2, zone.y + Math.floor(zone.height / 2) - 3);
        gfx.print(`${zone.label} ${Math.round(val * 100)}%`);
        break;
      }
      case 'sparkline': {
        const values = zone.value as number[];
        if (values.length < 2) break;
        const max = Math.max(...values);
        const min = Math.min(...values);
        const range = max - min || 1;

        const stepX = (zone.width - 4) / (values.length - 1);
        for (let i = 1; i < values.length; i++) {
          const x0 = zone.x + 2 + (i - 1) * stepX;
          const y0 = zone.y + zone.height - 2 -
            ((values[i - 1] - min) / range) * (zone.height - 4);
          const x1 = zone.x + 2 + i * stepX;
          const y1 = zone.y + zone.height - 2 -
            ((values[i] - min) / range) * (zone.height - 4);
          gfx.drawLine(
            Math.round(x0), Math.round(y0),
            Math.round(x1), Math.round(y1),
            zone.color
          );
        }
        break;
      }
    }
  }
}

// Usage: system monitor dashboard on 128x64 display
const display = new PixelBuffer(128, 64);
renderDashboard(display, [
  { x: 0, y: 0, width: 64, height: 20, type: 'text',
    label: 'CPU', value: '67%', color: 0xFFE0 },
  { x: 64, y: 0, width: 64, height: 20, type: 'text',
    label: 'MEM', value: '4.2G', color: 0x07E0 },
  { x: 0, y: 20, width: 128, height: 12, type: 'bar',
    label: 'DISK', value: 0.73, color: 0xFD20 },
  { x: 0, y: 32, width: 128, height: 32, type: 'sparkline',
    label: 'NET', value: [10, 25, 15, 40, 35, 50, 45, 30, 55, 60, 42, 38],
    color: 0x07FF },
]);
```

### Example 9: SPI Command Sequence for ST7735 Init

```typescript
// Typical ST7735 initialization sequence — same commands a real display needs
// This can drive the VirtualDisplayController directly

function initST7735(display: VirtualDisplayController): void {
  // Software reset
  display.writeCommand(0x01);

  // Sleep out
  display.writeCommand(0x11);

  // Frame rate control (normal mode)
  display.writeCommand(0xB1);
  display.writeData(0x01); // RTNA
  display.writeData(0x2C); // Front porch
  display.writeData(0x2D); // Back porch

  // Display inversion control
  display.writeCommand(0xB4);
  display.writeData(0x07);

  // Power control 1
  display.writeCommand(0xC0);
  display.writeData(0xA2);
  display.writeData(0x02);
  display.writeData(0x84);

  // Power control 2
  display.writeCommand(0xC1);
  display.writeData(0xC5);

  // VCOM control
  display.writeCommand(0xC5);
  display.writeData(0x0A);
  display.writeData(0x00);

  // Memory access control (rotation)
  display.writeCommand(0x36);
  display.writeData(0xC8); // Row/Col exchange, RGB order

  // Color mode: 16-bit (RGB565)
  display.writeCommand(0x3A);
  display.writeData(0x05); // 16 bits per pixel

  // Normal display mode
  display.writeCommand(0x13);

  // Display on
  display.writeCommand(0x29);

  // Set column address (0 to 127)
  display.writeCommand(0x2A);
  display.writeData(0x00);
  display.writeData(0x00);
  display.writeData(0x00);
  display.writeData(0x7F);

  // Set row address (0 to 159)
  display.writeCommand(0x2B);
  display.writeData(0x00);
  display.writeData(0x00);
  display.writeData(0x00);
  display.writeData(0x9F);

  // Ready for pixel data
  display.writeCommand(0x2C);
}

// Fill screen with a color gradient
function fillGradient(display: VirtualDisplayController): void {
  display.writeCommand(0x2A); // CASET
  display.writeData(0x00); display.writeData(0x00);
  display.writeData(0x00); display.writeData(0x7F);

  display.writeCommand(0x2B); // RASET
  display.writeData(0x00); display.writeData(0x00);
  display.writeData(0x00); display.writeData(0x9F);

  display.writeCommand(0x2C); // RAMWR

  for (let y = 0; y < 160; y++) {
    for (let x = 0; x < 128; x++) {
      const r = Math.round((x / 128) * 31);
      const g = Math.round((y / 160) * 63);
      const b = 15;
      const color = (r << 11) | (g << 5) | b;
      display.writeData(color >> 8);   // High byte
      display.writeData(color & 0xFF); // Low byte
    }
  }
}
```

### Example 10: Monochrome SSD1306 Page-Mode Emulator

```typescript
// The SSD1306 uses a unique "page" addressing mode
// The display is divided into 8 pages of 128 columns
// Each byte represents 8 vertical pixels in a page

class VirtualSSD1306 {
  private buffer: Uint8Array; // 128 * 8 pages = 1024 bytes
  private displayOn = false;
  private contrast = 0xCF;
  private invertDisplay = false;
  private startLine = 0;
  private pageAddress = 0;
  private columnAddress = 0;
  private pageStart = 0;
  private pageEnd = 7;
  private colStart = 0;
  private colEnd = 127;

  private currentCommand: number | null = null;
  private paramIndex = 0;
  private params: number[] = [];

  readonly width = 128;
  readonly height = 64;

  private listeners: ((buffer: Uint8Array) => void)[] = [];

  constructor() {
    this.buffer = new Uint8Array(1024); // 128 * 8
  }

  onUpdate(listener: (buffer: Uint8Array) => void): void {
    this.listeners.push(listener);
  }

  writeCommand(cmd: number): void {
    // Single-byte commands
    if (cmd >= 0xB0 && cmd <= 0xB7) {
      this.pageAddress = cmd & 0x07;
      return;
    }

    if ((cmd & 0xF0) === 0x00) {
      this.columnAddress = (this.columnAddress & 0xF0) | (cmd & 0x0F);
      return;
    }

    if ((cmd & 0xF0) === 0x10) {
      this.columnAddress = (this.columnAddress & 0x0F) | ((cmd & 0x0F) << 4);
      return;
    }

    switch (cmd) {
      case 0xAE: this.displayOn = false; break;
      case 0xAF: this.displayOn = true; this.emit(); break;
      case 0xA6: this.invertDisplay = false; this.emit(); break;
      case 0xA7: this.invertDisplay = true; this.emit(); break;
      case 0xE3: break; // NOP

      // Multi-byte commands
      case 0x81: // Set contrast
      case 0xD5: // Set display clock
      case 0xA8: // Set multiplex ratio
      case 0xD3: // Set display offset
      case 0xDA: // Set COM pins config
      case 0x8D: // Charge pump setting
      case 0x20: // Set memory addressing mode
        this.currentCommand = cmd;
        this.paramIndex = 0;
        this.params = [];
        break;

      case 0x21: // Set column address (2 params)
        this.currentCommand = cmd;
        this.paramIndex = 0;
        this.params = [];
        break;

      case 0x22: // Set page address (2 params)
        this.currentCommand = cmd;
        this.paramIndex = 0;
        this.params = [];
        break;
    }
  }

  writeData(data: number): void {
    if (this.currentCommand !== null) {
      this.params.push(data);

      switch (this.currentCommand) {
        case 0x81:
          this.contrast = data;
          this.currentCommand = null;
          break;
        case 0x21:
          if (this.params.length === 2) {
            this.colStart = this.params[0];
            this.colEnd = this.params[1];
            this.columnAddress = this.colStart;
            this.currentCommand = null;
          }
          break;
        case 0x22:
          if (this.params.length === 2) {
            this.pageStart = this.params[0];
            this.pageEnd = this.params[1];
            this.pageAddress = this.pageStart;
            this.currentCommand = null;
          }
          break;
        default:
          this.currentCommand = null;
          break;
      }
      return;
    }

    // Write to display buffer
    const idx = this.pageAddress * 128 + this.columnAddress;
    if (idx < this.buffer.length) {
      this.buffer[idx] = data;
    }

    this.columnAddress++;
    if (this.columnAddress > this.colEnd) {
      this.columnAddress = this.colStart;
      this.pageAddress++;
      if (this.pageAddress > this.pageEnd) {
        this.pageAddress = this.pageStart;
        this.emit();
      }
    }
  }

  // Convert page-mode buffer to pixel array
  toPixelArray(): Uint8Array {
    const pixels = new Uint8Array(this.width * this.height);
    for (let page = 0; page < 8; page++) {
      for (let col = 0; col < 128; col++) {
        const byte = this.buffer[page * 128 + col];
        for (let bit = 0; bit < 8; bit++) {
          const y = page * 8 + bit;
          const on = !!(byte & (1 << bit));
          pixels[y * this.width + col] = (on !== this.invertDisplay) ? 1 : 0;
        }
      }
    }
    return pixels;
  }

  private emit(): void {
    for (const listener of this.listeners) {
      listener(this.buffer);
    }
  }
}
```

---

## Comparisons

### Display Controller Comparison

| Controller | Type | Resolution | Colors | Interface | Command Protocol | Typical Use |
|-----------|------|-----------|--------|-----------|-----------------|-------------|
| **HD44780** | Character LCD | 16x2 to 40x4 chars | Mono (backlight color) | Parallel 4/8-bit | Instruction register + Data register | Status displays, appliances, DIY |
| **SSD1306** | Monochrome OLED | 128x64 | 1-bit (white/blue/yellow) | I2C / SPI | Page-mode addressing, column/row commands | Wearables, small status displays |
| **SSD1331** | Color OLED | 96x64 | 65K (RGB565) | SPI | CASET/RASET/RAMWR window addressing | Small color indicators, IoT |
| **ST7735** | Color TFT | 128x160 | 65K (RGB565) | SPI | CASET/RASET/RAMWR (same as ILI9341) | Hobby projects, portable devices |
| **ILI9341** | Color TFT | 240x320 | 262K (RGB666) | SPI / Parallel | CASET/RASET/RAMWR + MADCTL rotation | Dashboard displays, touchscreen UIs |
| **WS2812B** | Addressable LED | Any matrix | 16.7M (RGB888) | Single-wire timing protocol | Timed pulse sequences (no register model) | LED art, signage, decorative |
| **HUB75** | LED Panel | 32x16 to 128x64 | Varies (BCM depth) | Parallel shift-register | Row scanning + shift register clocking | Large LED signs, stadium displays |

### Graphics Library Comparison

| Library | Language | Display Support | Graphics API | Simulator? | Protocol | Best For |
|---------|----------|----------------|-------------|-----------|----------|---------|
| **[Adafruit GFX](https://github.com/adafruit/Adafruit-GFX-Library)** | C++ (Arduino) | 50+ display drivers | drawPixel + derived primitives | No (hardware only) | Direct GPIO/SPI/I2C | Arduino projects with any display |
| **[FastLED](https://github.com/FastLED/FastLED)** | C++ (Arduino) | WS2812, APA102, etc. | CRGB array, HSV native | Yes (via [Wokwi](https://wokwi.com/)) | Single-wire / SPI | LED strips, NeoPixel matrices |
| **[SmartMatrix](https://github.com/pixelmatix/SmartMatrix)** | C++ (Teensy/ESP32) | HUB75 panels | Layers + GFX-compatible | No | HUB75 shift-register | Large RGB LED panels |
| **[LVGL](https://lvgl.io/)** | C (WASM port) | Any display driver | Widgets, layouts, animations | Yes (browser via WASM) | Flush callback | Rich embedded UIs |
| **[WLED](https://kno.wled.ge/)** | C++ (ESP) | WS2812, SK6812, etc. | JSON API, 180+ effects | Yes (browser preview) | HTTP/WebSocket/MQTT | WiFi-controlled LED installations |
| **[Pixelblaze](https://www.bhencke.com/pixelblaze)** | Custom (JS-like) | WS2812, APA102, etc. | Per-pixel shader functions | Yes (built-in web editor) | WebSocket JSON | Creative LED art, live coding |
| **[led-matrix (npm)](https://www.npmjs.com/package/led-matrix)** | TypeScript | Canvas simulation | Matrix array input | Yes (is a simulator) | JavaScript API | Browser-based LED visualization |
| **[Pixlet (Tidbyt)](https://github.com/tidbyt/pixlet)** | Starlark (Python-like) | 64x32 RGB matrix | Widget-based (Text, Image, Plot) | Yes (browser + CLI) | WebP push | Tidbyt pixel display apps |
| **[RGBMatrixEmulator](https://github.com/ty-porter/RGBMatrixEmulator)** | Python | rpi-rgb-led-matrix compat | Same as rpi-rgb-led-matrix | Yes (is an emulator) | Drop-in Python module | Developing Raspberry Pi LED projects on desktop |

### LED Matrix Simulator Comparison

| Simulator | Platform | Input Format | Display Type | Multi-color | Open Source | URL |
|-----------|----------|-------------|-------------|-------------|-------------|-----|
| **[Wokwi](https://wokwi.com/)** | Browser | Arduino/ESP32 code | Full circuit simulation | Yes | No (free to use) | wokwi.com |
| **[RGBMatrixEmulator](https://github.com/ty-porter/RGBMatrixEmulator)** | Python (browser display) | rpi-rgb-led-matrix API | LED dot matrix | Yes | Yes (MIT) | GitHub |
| **[PixSim](https://www.henryhale.dev/pixsim/)** | Browser | Custom assembly language | LED matrix | Mono | Yes | henryhale.dev |
| **[led-matrix-simulator](https://github.com/sallar/led-matrix-simulator)** | Browser (HTML5 Canvas) | Pixel matrix array | LED dot matrix | Mono | Yes (MIT) | GitHub |
| **[Pixelique](https://pixelique.fun/)** | Browser | FastLED patterns | LED matrix | Yes | Partial | pixelique.fun |
| **[LED Matrix Editor](https://xantorohara.github.io/led-matrix-editor/)** | Browser | Click to toggle | 8x8 bitmap editor | Mono | Yes | GitHub |
| **[LVGL Simulator](https://sim.lvgl.io/)** | Browser (WASM) | C code compiled to WASM | Any display size | Yes | Yes | sim.lvgl.io |

### Protocol Adapter Comparison

| Protocol | Transport | Format | Bidirectional | Latency | Complexity | Best For |
|----------|-----------|--------|--------------|---------|------------|---------|
| **HD44780 direct** | Parallel GPIO (virtual) | Binary bytes | Yes (read busy flag) | ~37 us (virtual: instant) | Low | Character LCD emulation |
| **SPI command/data** | SPI bus (virtual) | Binary with DC pin | Half-duplex | ~ns (virtual: instant) | Medium | Color display emulation |
| **[LCDproc](https://lcdproc.org/)** | TCP (port 13666) | ASCII text commands | Yes | ~ms | Low | System monitoring displays |
| **[WLED JSON](https://kno.wled.ge/interfaces/json-api/)** | HTTP / WebSocket | JSON | Yes (WebSocket) | ~10-50ms | Medium | Remote LED control |
| **Serial/UART** | Serial port | Binary (with 0xFE prefix) | Half-duplex | Baud-dependent | Low | SparkFun SerLCD, serial backpacks |
| **Custom WebSocket** | WebSocket | JSON (GFX primitives) | Yes | ~1-5ms local | Medium | Browser-to-virtual display |
| **[Pixelblaze WS](https://www.bhencke.com/pixelblaze)** | WebSocket | JSON | Yes | ~10ms | Low | Live pattern control |

---

## Anti-Patterns

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Store display content as strings | Use DDRAM byte array like the real HD44780 | Strings can't represent custom CGRAM characters (codes 0-7), and you lose the address gap between rows |
| Implement cursor position as a simple `(x, y)` | Track as DDRAM address and derive row/col | The HD44780 address space is non-contiguous (row 2 starts at 0x40, not at column count). Direct x/y breaks when you implement display shift |
| Use RGB888 throughout for color displays | Use RGB565 internally, convert to RGB888 only at the Canvas render step | RGB565 matches what real displays use. Your virtual display should speak the same color format so protocol adapters work without conversion at every pixel |
| Re-render the entire display on every change | Dirty-rectangle tracking: only re-render cells that changed | A 64x32 RGB matrix at 60fps is fine. But a 240x320 ILI9341 at 60fps means 4.6M pixel operations per second. Track which address windows were written |
| Hardcode 16x2 display dimensions | Make dimensions configurable (16x2, 20x4, 40x2) | The HD44780 supports many sizes, and the DDRAM row offsets change based on column count. A 20x4 display has rows at offsets [0x00, 0x40, 0x14, 0x54] |
| Skip the initialization sequence | Require (or auto-run) the standard init sequence: Function Set, Display Control, Entry Mode, Clear | Real code sends init commands. If your virtual display works without them, protocol compatibility breaks — you'll accept malformed command streams without noticing |
| Use setTimeout for cursor blink | Use a state variable toggled by a single interval timer | Multiple setTimeout chains drift and create visual jitter. The HD44780 blinks at ~1.9 Hz (530ms period) — one interval is cleaner |
| Ignore the Entry Mode "shift" flag | Implement display shift on character write | When Entry Mode S=1, writing a character shifts the entire display. This is how scrolling marquee text works on real HD44780s. Skipping it breaks a common usage pattern |
| Convert RGB565 by just shifting bits | Fill the lower bits: `r = (r5 << 3) \| (r5 >> 2)` | Simple shifting gives max red of 248, not 255. You need to replicate the upper bits into the lower positions for full range |
| Build one monolithic display class | Separate state machine (HD44780), pixel buffer (GFX), and renderer (Canvas) | You want to swap renderers (Canvas, SwiftUI, terminal), connect different state machines (HD44780, SSD1306), and add protocol adapters (WebSocket, LCDproc) independently |

---

## HD44780 Complete Instruction Reference

For reference, here is the complete HD44780 instruction set in one table. This is the specification your virtual HD44780 must implement.

| Instruction | RS | RW | DB7 | DB6 | DB5 | DB4 | DB3 | DB2 | DB1 | DB0 | Execution Time | Description |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Clear Display | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | 1.52 ms | Clear all DDRAM, set address to 0, reset shift |
| Return Home | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | x | 1.52 ms | Set DDRAM address to 0, reset display shift |
| Entry Mode Set | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | I/D | S | 37 us | I/D=1: increment, S=1: shift display on write |
| Display Control | 0 | 0 | 0 | 0 | 0 | 0 | 1 | D | C | B | 37 us | D=display, C=cursor, B=blink |
| Cursor/Display Shift | 0 | 0 | 0 | 0 | 0 | 1 | S/C | R/L | x | x | 37 us | S/C=1: display shift, R/L=1: right |
| Function Set | 0 | 0 | 0 | 0 | 1 | DL | N | F | x | x | 37 us | DL=8-bit, N=2-line, F=5x10 font |
| Set CGRAM Addr | 0 | 0 | 0 | 1 | A5 | A4 | A3 | A2 | A1 | A0 | 37 us | Set CGRAM address (6-bit) |
| Set DDRAM Addr | 0 | 0 | 1 | A6 | A5 | A4 | A3 | A2 | A1 | A0 | 37 us | Set DDRAM address (7-bit) |
| Read Busy Flag | 0 | 1 | BF | AC6 | AC5 | AC4 | AC3 | AC2 | AC1 | AC0 | 0 | BF=1: busy, AC=address counter |
| Write Data | 1 | 0 | D7 | D6 | D5 | D4 | D3 | D2 | D1 | D0 | 37 us | Write to DDRAM or CGRAM |
| Read Data | 1 | 1 | D7 | D6 | D5 | D4 | D3 | D2 | D1 | D0 | 37 us | Read from DDRAM or CGRAM |

**DDRAM Address Map (common configurations):**

```
16x2:   Row 0: 0x00-0x0F    Row 1: 0x40-0x4F
20x2:   Row 0: 0x00-0x13    Row 1: 0x40-0x53
20x4:   Row 0: 0x00-0x13    Row 1: 0x40-0x53    Row 2: 0x14-0x27    Row 3: 0x54-0x67
40x2:   Row 0: 0x00-0x27    Row 1: 0x40-0x67
```

**CGRAM Map:**

```
Character 0: addresses 0x00-0x07  (8 rows, 5 bits each)
Character 1: addresses 0x08-0x0F
Character 2: addresses 0x10-0x17
...
Character 7: addresses 0x38-0x3F
```

---

## Practical Display Patterns for a macOS Notch HUD

Based on what people actually display on small LED panels and color LCDs, here are the most useful patterns for a macOS notch HUD context:

### What Works at Notch Scale (~200x30 pixels visible)

| Pattern | Description | Colors Needed | Complexity |
|---------|------------|---------------|------------|
| Scrolling status text | "Building... 45% complete" scrolling left | 2-3 (status color + bg) | Low |
| Progress bar | Thin horizontal bar with percentage | 2-3 (bar color varies by progress) | Low |
| System stats row | `CPU 45% MEM 67% DISK 73%` with color coding | 3-4 (green/yellow/red + white text) | Medium |
| Agent status icons | Row of 8x8 sprites: check, spinner, warning | 3 (green/cyan/yellow) | Low |
| Mini sparkline | 30-pixel-wide sparkline of last N values | 2 (line color + bg) | Low |
| Clock + weather | `14:32 72F` with weather icon | 2-3 | Medium |
| Build pipeline | Colored dots representing CI stages | 4+ (per-stage colors) | Low |
| Notification toast | Brief colored text that fades out | 2 (notification color + bg) | Medium |
| Stock ticker | Scrolling symbols with green/red changes | 3 (white/green/red) | Medium |
| Terminal-style | Fixed-width character grid like a real LCD | 2 (green on black) | Low |

### Color Recommendations for Dark Backgrounds

| Color | RGB565 | RGB888 | Use Case |
|-------|--------|--------|----------|
| LCD Green | 0x07E0 | #00FF00 | Classic status text, "all OK" |
| Amber | 0xFE00 | #FF7F00 | Warnings, retro LCD look |
| Cyan | 0x07FF | #00FFFF | Active/working state |
| Soft White | 0xC618 | #C0C0C0 | Labels, secondary text |
| Red | 0xF800 | #FF0000 | Errors, critical alerts |
| Yellow | 0xFFE0 | #FFFF00 | Warnings, attention needed |
| Blue | 0x001F | #0000FF | Links, info indicators |
| Dim Green | 0x0320 | #006400 | Inactive/idle state |

---

## Building a Complete Virtual Display Stack

Here's how all the pieces fit together for a macOS notch HUD:

```
                    +-----------------------+
                    |   macOS Canvas/SwiftUI |  <-- DotMatrixRenderer
                    |   (pixel rendering)   |      renders pixel buffer
                    +-----------+-----------+      to screen
                                |
                    +-----------+-----------+
                    |     PixelBuffer       |  <-- Stores RGB565 pixels
                    |   (graphics state)    |      implements GFX primitives
                    +-----------+-----------+
                                |
              +-----------------+------------------+
              |                 |                  |
   +----------+---+   +--------+------+   +-------+--------+
   | VirtualHD44780|  |VirtualDisplay |   | ScrollingText  |
   | (char LCD    |   |Controller     |   | Engine         |
   |  state)      |   |(color SPI)    |   | (animation)    |
   +-----------+--+   +--------+------+   +-------+--------+
               |               |                   |
   +-----------+--+   +--------+------+   +--------+-------+
   |HD44780Canvas |   |SPI Command    |   | Direct GFX     |
   |Bridge        |   |Adapter        |   | API calls      |
   +-----------+--+   +--------+------+   +--------+-------+
               |               |                   |
   +-----------+---------------+-------------------+
   |              WebSocket Protocol Server         |
   |    (accepts HD44780 / SPI / GFX / LCDproc)    |
   +-----------------------------------------------+
               |
     Remote processes (CLI tools, monitoring agents,
     build systems, AI agents) send display commands
```

The key architectural insight: separate the **state machine** (what data is in the display), the **pixel buffer** (what pixels are lit), and the **renderer** (how pixels become visible). Each layer can be swapped independently:

- Swap the state machine: HD44780 for character mode, SSD1306 for mono OLED, ILI9341 for color TFT
- Swap the renderer: HTML Canvas for web, Core Graphics for native macOS, SwiftUI for Notchy integration
- Add protocol adapters: WebSocket for remote control, LCDproc for system monitoring, WLED for LED ecosystem tools

---

## Swift/macOS Integration Notes

For integration with a macOS notch app like Notchy, the virtual display stack maps to Swift as follows:

```
TypeScript PixelBuffer  -->  Swift class with [UInt16] buffer
TypeScript DotMatrixRenderer  -->  NSView.draw() or SwiftUI Canvas
TypeScript VirtualHD44780  -->  Swift class (same state machine logic)
TypeScript WebSocket server  -->  Network.framework NWListener
```

The rendering path in Swift would use `CGContext` or `SwiftUI Canvas`:

```swift
// Swift equivalent of DotMatrixRenderer (conceptual)
struct DotMatrixView: View {
    let pixels: [UInt16]  // RGB565 buffer
    let gridWidth: Int
    let gridHeight: Int
    let dotSize: CGFloat = 3.0
    let gap: CGFloat = 1.0

    var body: some View {
        Canvas { context, size in
            for y in 0..<gridHeight {
                for x in 0..<gridWidth {
                    let color = pixels[y * gridWidth + x]
                    let r = Double((color >> 11) & 0x1F) / 31.0
                    let g = Double((color >> 5) & 0x3F) / 63.0
                    let b = Double(color & 0x1F) / 31.0

                    let rect = CGRect(
                        x: CGFloat(x) * (dotSize + gap),
                        y: CGFloat(y) * (dotSize + gap),
                        width: dotSize,
                        height: dotSize
                    )

                    context.fill(
                        Path(ellipseIn: rect),
                        with: .color(Color(
                            red: r, green: g, blue: b
                        ))
                    )
                }
            }
        }
    }
}
```

For network communication, use `Network.framework` to accept WebSocket connections on a local port, enabling CLI tools and background processes to push content to the notch display:

```swift
// Conceptual WebSocket server for display commands
import Network

let listener = try NWListener(using: .tcp, on: 13666)
listener.newConnectionHandler = { connection in
    // Upgrade to WebSocket, parse JSON messages,
    // route to VirtualHD44780 or PixelBuffer
}
listener.start(queue: .main)
```

---

## References

### Official Documentation & Datasheets

- [Hitachi HD44780 LCD Controller — Wikipedia](https://en.wikipedia.org/wiki/Hitachi_HD44780_LCD_controller) — Comprehensive reference for the HD44780 instruction set, DDRAM layout, CGRAM, and 4-bit/8-bit modes
- [HD44780 Instruction Set — Andrew Dawes](https://dawes.wordpress.com/2010/01/05/hd44780-instruction-set/) — Concise command reference with hex values for common operations
- [HD44780 Commands — Dincer Aydin](http://dinceraydin.com/lcd/commands.htm) — Complete command table with timing information
- [Programming the HD44780 — Glenn Klockwood](https://www.glennklockwood.com/electronics/hd44780-lcd-display.html) — Practical guide to HD44780 register addressing and initialization
- [HD44780 LCD Initialization — Alfred State](https://web.alfredstate.edu/faculty/weimandn/lcd/lcd_initialization/lcd_initialization_index.html) — Detailed initialization sequence with timing requirements
- [HD44780 — elm-chan.org](https://elm-chan.org/docs/lcd/hd44780_e.html) — Technical reference with pin diagrams and bus timing

### Graphics & Display Libraries

- [Adafruit GFX Library — GitHub](https://github.com/adafruit/Adafruit-GFX-Library) — Core graphics primitive library for Arduino displays; the standard API that all other libraries derive from
- [Adafruit GFX Graphics Primitives — Learning Guide](https://learn.adafruit.com/adafruit-gfx-graphics-library/graphics-primitives) — Official documentation of drawPixel, drawLine, drawRect, drawCircle, and text rendering functions
- [FastLED — GitHub](https://github.com/FastLED/FastLED) — High-performance LED animation library supporting WS2812B, APA102, and 50+ other LED chipsets
- [FastLED_NeoMatrix — GitHub](https://github.com/marcmerlin/FastLED_NeoMatrix) — Adafruit GFX-compatible wrapper for using FastLED with NeoPixel matrices
- [SmartMatrix — GitHub](https://github.com/pixelmatix/SmartMatrix) — HUB75 LED panel driver for Teensy and ESP32 with high color depth via Binary Code Modulation
- [ESP32-HUB75-MatrixPanel-DMA — GitHub](https://github.com/mrcodetastic/ESP32-HUB75-MatrixPanel-DMA) — ESP32 HUB75 driver using DMA for high refresh rates; Adafruit GFX compatible
- [LVGL — Light and Versatile Graphics Library](https://lvgl.io/) — Embedded graphics library with 30+ widgets, layout managers, and WebAssembly support for browser simulation
- [LVGL Color Format Documentation](https://docs.lvgl.io/master/main-modules/display/color_format.html) — RGB565, RGB888, and other color format specifications

### LED Control Protocols & Ecosystems

- [WLED JSON API](https://kno.wled.ge/interfaces/json-api/) — HTTP/WebSocket JSON API for controlling WS2812 LED installations; supports segments, effects, and palettes
- [WLED WebSocket Interface](https://kno.wled.ge/interfaces/websocket/) — Real-time WebSocket protocol for WLED devices at /ws endpoint
- [WLED HTTP API](https://kno.wled.ge/interfaces/http-api/) — Simple GET-based API for basic WLED control
- [Pixelblaze V3 — Crowd Supply](https://www.crowdsupply.com/hencke-technologies/pixelblaze-v3) — WiFi LED controller with built-in JavaScript-like pattern language and WebSocket API
- [pixelblaze-client — PyPI](https://pypi.org/project/pixelblaze-client/) — Python client for Pixelblaze WebSocket API
- [Pixlet (Tidbyt) — GitHub](https://github.com/tidbyt/pixlet) — Starlark-based app runtime for 64x32 pixel displays; renders to WebP

### LED Matrix Simulators

- [RGBMatrixEmulator — GitHub](https://github.com/ty-porter/RGBMatrixEmulator) — Python desktop emulator for rpi-rgb-led-matrix; renders in browser
- [Emulating Raspberry Pi LED Matrices in Your Browser — Tyler Porter](https://blog.ty-porter.dev/development/emulation/2022/06/13/browser_emulation_rpi_led_matrix.html) — Blog post on building browser-based LED matrix emulation
- [led-matrix-simulator — GitHub](https://github.com/sallar/led-matrix-simulator) — Simple HTML5 Canvas LED matrix simulator
- [led-matrix (npm)](https://www.npmjs.com/package/led-matrix) — HTML5 Canvas LED Matrix simulator accepting pixel matrix arrays
- [PixSim — Henry Hale](https://www.henryhale.dev/pixsim/) — Browser-based LED matrix simulator with custom assembly language
- [DMD-Simulator — GitHub](https://github.com/MysteriousBits/DMD-Simulator) — Web app simulating 64x16 LED dot matrix displays
- [Wokwi Arduino Simulator](https://wokwi.com/) — Full Arduino/ESP32 circuit simulator with LED matrix and LCD display support
- [LED Matrix Editor — Xantorohara](https://xantorohara.github.io/led-matrix-editor/) — Browser-based 8x8 LED matrix bitmap editor
- [Pixelique — FastLED Animations Editor](https://pixelique.fun/) — Browser-based FastLED pattern editor and LED matrix simulator

### Display Controllers & Hardware

- [SSD1306 Driver Library — GitHub (lexus2k)](https://github.com/lexus2k/ssd1306) — Multi-controller driver supporting SSD1306, SSD1331, SSD1351, ILI9341, ST7735
- [Luma.OLED Hardware Documentation](https://luma-oled.readthedocs.io/en/latest/hardware.html) — Python drivers for SSD1306, SSD1331, SSD1351, and other OLED controllers
- [rpi-rgb-led-matrix — GitHub (hzeller)](https://github.com/hzeller/rpi-rgb-led-matrix) — Definitive Raspberry Pi HUB75 LED matrix library; the standard that emulators target
- [HUB75 Panel Guide — SmartMatrix Wiki](https://github.com/pixelmatix/SmartMatrix/wiki/HUB75-Panels) — Technical reference for HUB75 LED panel configurations and compatibility

### Color Formats & Conversion

- [RGB565 to RGB888 Color Conversion — SheekGeek](https://sheekgeek.org/2020/adamsheekgeek/rgb565-to-rgb888-color-conversion) — Practical guide to bit manipulation for RGB565/RGB888 conversion
- [About RGB565 — Barth Development](https://barth-dev.de/about-rgb565-and-how-to-convert-into-it/) — Explains why RGB565 exists and how to convert to/from it
- [RGB565 Color Picker](https://rgbcolorpicker.com/565) — Interactive tool for picking RGB565 colors
- [RGB565 vs RGB888 — Forlinx](https://www.forlinx.net/industrial-news/difference-between-rgb565-and-rgb888-423.html) — Comparison of color formats with memory usage analysis

### Virtual Display Daemons

- [LCDproc — LCDd Daemon](https://www.mankier.com/8/LCDd) — Linux LCD display daemon on port 13666 with client-server protocol
- [LCDproc Protocol Specification](https://lcdproc.org/download/netstuff.txt) — ASCII text protocol: hello, screen_add, widget_set, etc.
- [LCDproc Protocol Reference — CPAN](https://metacpan.org/dist/Net-LCDproc/view/doc/lcdproc_protocol.pod) — Perl documentation of LCDproc client-server protocol commands
- [lcd4linux — Man Page](https://www.mankier.com/8/lcd4linux) — Alternative LCD daemon that grabs kernel info and displays on external LCDs

### JavaScript/TypeScript Display Libraries

- [rpi-led-matrix (Node.js) — GitHub](https://github.com/alexeden/rpi-led-matrix) — TypeScript bindings for rpi-rgb-led-matrix with full type definitions
- [wled-client — GitHub](https://github.com/ShiftLimits/wled-client) — JavaScript/TypeScript client for WLED devices from Node.js or browser
- [DotMatrx.js — CSS Script](https://www.cssscript.com/interactive-dot-matrix/) — SVG-based interactive dot matrix display for web
- [awesome-ws2812 — GitHub](https://github.com/PabloCastellano/awesome-ws2812) — Curated list of WS2812 LED resources including software simulators

### Blog Posts & Tutorials

- [ESP32 NeoPixel Library Comparison — Jake's Blog](https://blog.ja-ke.tech/2019/06/02/neopixel-performance.html) — Performance benchmarks comparing FastLED, NeoPixelBus, and Adafruit NeoPixel
- [Drawing to the ILI9341 — Vivonomicon](https://vivonomicon.com/2018/06/17/drawing-to-a-small-tft-display-the-ili9341-and-stm32/) — Low-level SPI command walkthrough for ILI9341 initialization and pixel writing
- [Using an HD44780 LCD Screen — Gibbard](https://www.gibbard.me/hd44780_lcd_screen/) — Practical guide to HD44780 wiring, initialization, and DDRAM addressing
