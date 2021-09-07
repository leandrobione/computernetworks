### Usage
`man crunch`

`crunch <min-len> <max-len> [<charset string>] [options]`

### Example
`crunch 4 4 -t tx@%`

- @: characters
- %: numbers

### Pipeline
`crunch 12 12 -t passwd@@%%%% | aircrack-ng -w - [SCAN].cap -e [wlan]`
