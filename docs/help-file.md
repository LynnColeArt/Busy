# Busy Help Guide

## Overview
BusyParser is a flexible file processing utility designed to parse and transform `.op` files through a transaction-based approach.

## Installation
```bash
go install github.com/LynnColeArt/Busy
```

## Usage Patterns

### Single File Processing
```bash
busy file.op
```

### Multiple File Processing
```bash
busy file1.op file2.op file3.op
```

### Directory Processing
```bash
busy -dir ./operations
```

## Command Options

### `-v` (Verbose)
- Enables detailed output during processing
- Shows processing statistics and detailed information

### `-d` (Dry Run)
- Simulates processing without making actual changes
- Useful for validation and testing

### `-h` (Help)
- Displays comprehensive help information
- Shows usage patterns and available options

### `-dir` (Directory)
- Processes all `.op` files in specified directory
- Allows batch processing of multiple files

## .op File Format

### Transaction Structure
```
~ 
action: create
target: path/to/file
reasoning: Why this operation
developer-notes: Technical details
author: Your Name

[block]
Content goes here
[/block]
~
```

## Error Handling
- Comprehensive error tracking
- Prevents partial processing
- Provides detailed error context

## Best Practices
- Ensure complete transaction metadata
- Use clear, descriptive reasoning
- Validate .op files before batch processing

## Troubleshooting
- Check file extensions (.op)
- Verify transaction metadata completeness
- Review verbose output for detailed insights
