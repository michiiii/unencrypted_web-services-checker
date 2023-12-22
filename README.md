# URL HTTPS Redirection Checker

## Description

This script checks a list of HTTP URLs to determine if they are directly redirected to HTTPS. It captures details of the final URLs, their titles, and the presence of the HSTS (HTTP Strict Transport Security) header. The purpose is to help understand and audit the security posture of websites, specifically their implementation of HTTPS and HSTS.

## Features

- Checks for direct HTTPS redirection.
- Captures the title of the final landing page.
- Detects the presence of the HSTS header.
- Handles a list of URLs for bulk processing.
- Provides a progress bar for monitoring execution.

## Requirements

- Python 3.6+
- `aiohttp`
- `aiofiles`
- `BeautifulSoup4`
- `tqdm`

## Setup

1. Ensure you have Python 3.6+ installed on your system.
2. Install the required Python packages:

    ```bash
    pip install aiohttp aiofiles beautifulsoup4 tqdm
    ```

3. Place the script in your desired directory.

## Usage

1. Prepare a text file containing one URL per line that you wish to check. Ensure these URLs start with `http://`.
2. Run the script from the command line, specifying the input file with URLs and the desired output CSV file:

    ```bash
    python url_https_redirection_checker.py --in input_urls.txt --out results.csv
    ```

    - `--in`: Specifies the path to the input file containing the list of URLs.
    - `--out`: Specifies the path to the output CSV file where results will be saved.

3. The script will process each URL and output the results to the specified CSV file. The columns will include:
    - `URL`: The original URL checked.
    - `Status Code`: The HTTP status code received.
    - `Title`: The title of the final landing page (if any).
    - `HSTS Present`: Indicates whether the HSTS header is present ('Yes' or 'No').

## Debugging

The script includes print statements for logging the process. If you encounter any issues, check the console output for error messages and the URL causing the problem.

## Notes

- The script is designed for educational and auditing purposes. Do not use it for any unauthorized or illegal activities.
- Disabling SSL verification is generally not recommended for production environments due to security risks. It's used here for simplicity and should be handled more cautiously in a real-world scenario.

## License

Specify your license or state that the project is unlicensed.
