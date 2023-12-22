"""
URL HTTPS Redirection Checker

This script checks a list of HTTP URLs for direct HTTPS redirections and captures details of URLs, titles, and HSTS status at their final destination. It is designed to help understand the security posture of a list of websites by identifying which ones automatically upgrade their connections to HTTPS and which ones do not, as well as checking for the presence of the HSTS header.

Usage:
  - Place a list of HTTP URLs in a text file, one URL per line.
  - Run the script with the input file as --in and the desired output file as --out.
  - The script will process each URL and record the details in the specified CSV output file.
"""

import aiohttp
import asyncio
import aiofiles
from bs4 import BeautifulSoup
from tqdm.asyncio import tqdm
import ssl
import argparse

# Set up command-line argument parsing
parser = argparse.ArgumentParser(description='Check HTTP URLs for direct HTTPS redirection and capture details of URLs, titles, and HSTS status at final destination.')
parser.add_argument('--in', dest='input_file', required=True, help='File containing URLs to check (one per line).')
parser.add_argument('--out', dest='output_file', required=True, help='CSV file to write details of URLs, titles, and HSTS status.')
args = parser.parse_args()

async def get_title(session, url):
    """
    Asynchronously fetches the title and HSTS status of a given URL.

    :param session: The aiohttp client session.
    :param url: The URL to fetch.
    :return: A tuple containing the title and HSTS status.
    """
    try:
        async with session.get(url, timeout=10) as response:
            hsts = 'Yes' if 'Strict-Transport-Security' in response.headers else 'No'
            if response.status == 200:
                text = await response.text()
                soup = BeautifulSoup(text, 'html.parser')
                title = soup.find('title').text if soup.find('title') else 'No Title Found'
                return title, hsts
    except Exception:
        return 'Failed to retrieve title', 'No'

async def check_url(session, url, progress):
    """
    Checks a single URL for HTTPS redirection and captures relevant details.

    :param session: The aiohttp client session.
    :param url: The URL to check.
    :param progress: A tqdm progress bar object.
    :return: A tuple with URL, status code, title, and HSTS status or None.
    """
    try:
        if url.startswith('http://'):
            async with session.get(url, allow_redirects=False, timeout=2) as response:
                # Debugging log for each URL checked
                print(f"Checking URL: {url}")

                if response.status in (301, 302, 303, 307, 308):  # All common redirect status codes
                    location = response.headers.get('Location', '')
                    print(f"Redirected to: {location}")  # Debugging log

                    if not location:  # In case the location header is missing
                        return url, response.status, 'No Location Header', 'No'
                    
                    if location.startswith('https://'):
                        print(f"Excluding {url} as it redirects to HTTPS")  # Debugging log
                    else:  # Follow the redirect manually
                        title, hsts = await get_title(session, location)
                        progress.update(1)
                        return url, response.status, title, hsts
                else:  # Not a redirect, get the title directly
                    title, hsts = await get_title(session, url)
                    progress.update(1)
                    return url, response.status, title, hsts
    except Exception as e:
        progress.update(1)
        # Debugging log for any errors encountered
        print(f"Error processing {url}: {e}")
        return url, 'Error', str(e), 'No'

    progress.update(1)
    return None

async def main(input_file, output_file):
    """
    The main coroutine, which reads the list of URLs and processes each one.

    :param input_file: Path to the file containing the list of URLs.
    :param output_file: Path to the file where results will be saved.
    """
    # Read URLs from the input file
    async with aiofiles.open(input_file, mode='r') as file:
        urls = [line.strip() for line in await file.readlines()]

    # Create an SSL context to ignore certificate errors (not recommended for production)
    ssl_context = ssl.create_default_context()
    ssl_context.check_hostname = False
    ssl_context.verify_mode = ssl.CERT_NONE

    # Create an aiohttp session with the SSL context
    connector = aiohttp.TCPConnector(ssl=ssl_context)
    async with aiohttp.ClientSession(connector=connector) as session:
        progress = tqdm(total=len(urls), desc='Checking URLs')

        # Create tasks for each URL and process them concurrently
        tasks = [asyncio.ensure_future(check_url(session, url, progress)) for url in urls]
        results = await asyncio.gather(*tasks)
        progress.close()

        # Filter out None results and write to the output file
        results_to_record = [result for result in results if result is not None]
        async with aiofiles.open(output_file, mode='w', newline='') as csvfile:
            await csvfile.write('URL,Status Code,Title,HSTS Present\n')
            for site in results_to_record:
                await csvfile.write(f'{site[0]},{site[1]},{site[2]},{site[3]}\n')

    print(f"Script completed. Details, titles, and HSTS status of URLs have been recorded in '{output_file}'.")

# Run the main coroutine with the provided arguments
asyncio.run(main(args.input_file, args.output_file))