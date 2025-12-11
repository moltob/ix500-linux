# ix500-linux

One-button scanning workflow for Fujitsu ScanSnap iX500 on Linux.

Press the scanner button → get a searchable PDF with OCR. That's it.

## Features

- **One-button scanning** - press the hardware button, get a PDF
- **Duplex A4 color** - scans both sides automatically
- **Smart blank detection** - skips empty back pages (20% threshold)
- **Auto color/grayscale** - converts B&W pages to grayscale for smaller files
- **OCR** - searchable text with Dutch + English (configurable)
- **Auto-rotate** - fixes upside-down pages automatically
- **Image cleanup** - deskew, crop borders, remove speckles
- **Aggressive compression** - ~95% smaller than raw scans
- **PDF/A-2b output** - archival quality
- **Desktop notifications** - know when your scan is done
- **Robust disconnect handling** - no crashes when scanner is unplugged

## Requirements

### System packages
- `sane` / `sane-backends` - scanner driver (usually pre-installed)

### Homebrew packages
```bash
brew install ocrmypdf tesseract tesseract-lang
```

### Tesseract symlinks
Brew doesn't always link these automatically. Check if they exist and create if needed:
```bash
# Check your tesseract version
TESS_VERSION=$(brew list --versions tesseract | awk '{print $2}')

# Create symlinks if missing
ln -sf "/home/linuxbrew/.linuxbrew/Cellar/tesseract/${TESS_VERSION}/share/tessdata/eng.traineddata" /home/linuxbrew/.linuxbrew/share/tessdata/
ln -sf "/home/linuxbrew/.linuxbrew/Cellar/tesseract/${TESS_VERSION}/share/tessdata/osd.traineddata" /home/linuxbrew/.linuxbrew/share/tessdata/
```

## Installation

### 1. Verify scanner is detected
```bash
scanimage -L
# Should show: device `fujitsu:ScanSnap iX500:XXXXX'
```

### 2. Install scripts
```bash
# Copy scan scripts
cp scan ~/.local/bin/scan
cp scan-button-poll ~/.local/bin/scan-button-poll
chmod +x ~/.local/bin/scan ~/.local/bin/scan-button-poll

# Scripts auto-detect scanner - no manual configuration needed
```

### 3. Install systemd user service
```bash
cp scan-button.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable scan-button.service
```

### 4. Install udev rule (for auto start on USB connect)
```bash
sudo cp 99-scansnap-ix500.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 5. Start the service
```bash
systemctl --user start scan-button.service
```

## Usage

### One-button scanning
1. Put documents in the feeder
2. Press the blue Scan button on the scanner
3. Wait for desktop notification "Scan Complete"
4. Find your PDF in `~/Documents/scanner-inbox/`

### Manual scanning
```bash
scan                  # Auto-named: scan-YYYY-MM-DD-HHMMSS.pdf
scan my-document      # Custom name: my-document.pdf
```

### Check service status
```bash
systemctl --user status scan-button.service
journalctl --user -u scan-button.service -f
```

## Configuration

### Scanner options (in `scan`)

| Option | Value | Purpose |
|--------|-------|---------|
| `--swskip` | 20% | Skip blank pages |
| `--swcrop` | yes | Auto-crop borders |
| `--swdespeck` | 2 | Remove small artifacts |
| `--overscan` | On | Better feed handling |
| `--prepick` | On | Pre-pick next page (faster) |
| `--buffermode` | On | Faster processing |

### OCR options (in `scan`)

| Option | Value | Purpose |
|--------|-------|---------|
| `-l` | nld+eng | Languages (Dutch + English) |
| `--rotate-pages-threshold` | 2 | Aggressive rotation fix |
| `-O` | 3 | Maximum optimization |
| `--jpeg-quality` | 60 | Good compression |
| `--jbig2-lossy` | yes | Better text compression |

### Color detection
Pages with <10% color saturation are converted to grayscale automatically.

## How it works

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Button     │────▶│ scan-button  │────▶│ scan script     │
│  pressed    │     │ -poll        │     │                 │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                    ┌─────────────────────────────┘
                    ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  scanimage  │────▶│ ImageMagick  │────▶│ ocrmypdf        │
│  (SANE)     │     │ (combine)    │     │ (OCR + PDF)     │
└─────────────┘     └──────────────┘     └─────────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │ ~/Documents/    │
                                         │ scanner-inbox/  │
                                         └─────────────────┘
```

1. **Button poll service** checks scanner button every 500ms
2. **scanimage** captures duplex color TIFF pages
3. **ImageMagick** combines pages, detects color vs grayscale
4. **ocrmypdf** adds OCR layer, optimizes, outputs PDF/A-2b
5. **Desktop notification** confirms completion

## Tested on

- **OS**: Bluefin (Fedora Silverblue based)
- **Scanner**: Fujitsu ScanSnap iX500
- **SANE**: sane-backends with fujitsu driver

Should work on any Linux with SANE support for the iX500.

## Credits

- [Rida Ayed's ix500 Linux guide](https://ridaayed.com/posts/setup_fujitsu_ix500_scanner_linux/) - scanner options and swskip threshold
- [foxey/scanbdScanSnapIntegration](https://github.com/foxey/scanbdScanSnapIntegration) - inspiration for button daemon
- [OCRmyPDF](https://ocrmypdf.readthedocs.io/) - excellent OCR tool

## License

MIT
