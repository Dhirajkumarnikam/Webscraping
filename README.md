import requests
from bs4 import BeautifulSoup
import csv
import time

# Function to scrape product details from a listing page
def scrape_listing_page(url):
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
    }

    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, 'html.parser')

    products = soup.find_all('div', {'data-component-type': 's-search-result'})
    product_details = []

    for product in products:
        product_url = "https://www.amazon.in" + product.find('a', {'class': 'a-link-normal'})['href']
        product_name = product.find('span', {'class': 'a-text-normal'}).text.strip()
        product_price = product.find('span', {'class': 'a-price-whole'}).text.strip()

        rating_tag = product.find('span', {'class': 'a-icon-alt'})
        rating = rating_tag.text.split()[0] if rating_tag else 'N/A'

        num_reviews_tag = product.find('span', {'class': 'a-size-base'})
        num_reviews = num_reviews_tag.text.strip() if num_reviews_tag else 'N/A'

        product_details.append({
            "Product URL": product_url,
            "Product Name": product_name,
            "Product Price": product_price,
            "Rating": rating,
            "Number of Reviews": num_reviews
        })

    return product_details

# Function to scrape additional product details from a product page
def scrape_product_page(url):
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
    }

    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, 'html.parser')

    product_description_tag = soup.find('div', {'id': 'productDescription'})
    product_description = product_description_tag.get_text(strip=True) if product_description_tag else 'N/A'

    manufacturer_tag = soup.find('a', {'id': 'bylineInfo'})
    manufacturer = manufacturer_tag.get_text(strip=True) if manufacturer_tag else 'N/A'

    asin = url.split('/')[-1].split('?')[0]

    description = soup.find('meta', {'name': 'description'})['content'] if soup.find('meta', {'name': 'description'}) else 'N/A'

    return {
        "ASIN": asin,
        "Product Description": product_description,
        "Manufacturer": manufacturer,
        "Description": description
    }

# Main function
def main():
    base_url = "https://www.amazon.in/s"
    search_params = {
        "k": "bags",
        "crid": "2M096C61O4MLT",
        "qid": "1653308124",
        "sprefix": "ba,aps,283",
        "ref": "sr_pg_1"
    }

    num_pages_to_scrape = 20
    product_urls = []
    product_details = []

    # Scraping product URLs from listing pages
    for page in range(1, num_pages_to_scrape + 1):
        search_params["page"] = page
        response = requests.get(base_url, params=search_params)
        soup = BeautifulSoup(response.text, 'html.parser')

        products = soup.find_all('div', {'data-component-type': 's-search-result'})
        for product in products:
            product_url = "https://www.amazon.in" + product.find('a', {'class': 'a-link-normal'})['href']
            product_urls.append(product_url)

    # Scraping product details from individual product pages
    for product_url in product_urls:
        product_detail = scrape_product_page(product_url)
        product_details.append(product_detail)

        # Sleep to avoid rapid requests and respect the website's terms
        time.sleep(1)

    # Exporting data to CSV
    csv_filename = 'amazon_product_data.csv'
    csv_header = ["Product URL", "Product Name", "Product Price", "Rating", "Number of Reviews", "ASIN", "Product Description", "Manufacturer", "Description"]

    with open(csv_filename, 'w', newline='', encoding='utf-8') as csv_file:
        csv_writer = csv.DictWriter(csv_file, fieldnames=csv_header)
        csv_writer.writeheader()

        for i in range(len(product_details)):
            product_detail = product_details[i]
            product_detail["Product URL"] = product_urls[i]
            csv_writer.writerow(product_detail)

    print("Data exported to", csv_filename)

if __name__ == "__main__":
    main()

