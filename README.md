# AI Skills Repository

A collection of AI-powered skills and commands for the Kilo assistant framework, along with software development documentation.

## Project Structure

```
ai-skills/
├── .kilo/
│   ├── command/          # Custom Kilo commands
│   └── skills/           # Custom Kilo skills
├── create-fsd/           # FSD/TSD document creation skill
│   ├── references/       # Review rubrics and templates
│   └── results/          # Generated documentation samples
├── kilo.json             # Kilo configuration
└── package.json          # Node.js dependencies
```

## Features

### Kilo Skills

| Skill | Description |
|-------|-------------|
| `create-fsd` | Creates review-ready Functional Specification Design (FSD) and Technical Specification Design (TSD) documents |
| `analyse-fsd` | Reviews architecture and development documents across multiple formats |
| `analysis-cve` | Contextual CVE/CVSS analysis for vulnerability threat assessment |
| `review-log` | Analyzes application logs to find errors, patterns, and root causes |
| `kilo-config` | Kilo configuration guide and Agent Manager setup |

### Documentation Templates

- **FSD Review Rubric** - Checklist for Functional Specification Design reviews
- **TSD Review Rubric** - Checklist for Technical Specification Design reviews
- **Methodology Matrix** - Agile/SAFe/Waterfall compliance criteria
- **Output Templates** - Tech Lead, VP, and Board-level review formats

## Usage

### Using Skills with Kilo

```bash
# Create an FSD document
/create-fsd "Your project description here"

# Analyze an existing FSD
/analyse-fsd @path/to/your/document.md

# Review application logs
/review-log "Error logs from the application"
```

### Running Commands

Commands are located in `.kilo/command/` and can be invoked directly through the Kilo interface.

## Sample Documents

The `create-fsd/results/` directory contains sample FSD documents generated for various projects:

- **SAMAHI** - Sistem Manajemen Aset dan Monitoring Infrastruktur (Infrastructure Asset and Monitoring Management System)
- Multiple AI-generated FSD samples from different models (ChatGPT, Qwen, Gemini)

## Requirements

- Node.js (for dependencies)
- Kilo assistant installed
- Git for version control

## Installation

```bash
git clone https://github.com/javajammer/ai-skills.git
cd ai-skills
npm install
```

## Configuration

Edit `kilo.json` to customize:
- Skill paths
- Command paths
- Provider settings
- TUI preferences

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project is proprietary and confidential.

## Contact

- **Repository**: [javajammer/ai-skills](https://github.com/javajammer/ai-skills)
- **Maintainer**: javajammer