# -Web-Scraping
Web Scraping
#include <stdio.h>
#include <string.h>
#include <curl/curl.h>
#include <libxml/HTMLparser.h>
#include <libxml/xpath.h>

#define MAX_PRODUCTS 10  // Define the maximum number of products to scrape
#define MAX_STRING_LENGTH 100

struct Product {
    char name[MAX_STRING_LENGTH];
    char price[MAX_STRING_LENGTH];
    char rating[MAX_STRING_LENGTH];
};

struct MemoryStruct {
    char *memory;
    size_t size;
};

// Callback function to handle HTTP response
size_t WriteMemoryCallback(void *contents, size_t size, size_t nmemb, void *userp) {
    size_t realsize = size * nmemb;
    struct MemoryStruct *mem = (struct MemoryStruct *)userp;

    char *ptr = realloc(mem->memory, mem->size + realsize);
    if (ptr == NULL) {
        return 0;  // Out of memory
    }

    mem->memory = ptr;
    memcpy(&(mem->memory[mem->size]), contents, realsize);
    mem->size += realsize;

    return realsize;
}

int main() {
    CURL *curl;
    CURLcode res;
    struct MemoryStruct chunk;

    chunk.memory = NULL;
    chunk.size = 0;

    curl = curl_easy_init();
    if (curl) {
        curl_easy_setopt(curl, CURLOPT_URL, "https://example.com/products");  // Replace with the URL of the e-commerce website
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteMemoryCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, (void *)&chunk);

        res = curl_easy_perform(curl);

        if (res != CURLE_OK) {
            fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
        } else {
            // Parse the HTML content with libxml2
            htmlDocPtr doc = htmlReadMemory(chunk.memory, chunk.size, "noname", NULL, HTML_PARSE_RECOVER | HTML_PARSE_NOERROR | HTML_PARSE_NOWARNING);
            if (doc != NULL) {
                xmlDocPtr xmlDoc = doc;
                xmlXPathContextPtr context = xmlXPathNewContext(xmlDoc);

                if (context != NULL) {
                    xmlXPathObjectPtr productNodes = xmlXPathEvalExpression(BAD_CAST("//div[@class='product-item']"), context);

                    if (productNodes != NULL) {
                        xmlNodeSetPtr nodeset = productNodes->nodesetval;
                        int i;
                        struct Product products[MAX_PRODUCTS];

                        for (i = 0; i < nodeset->nodeNr && i < MAX_PRODUCTS; i++) {
                            xmlNodePtr node = nodeset->nodeTab[i];
                            xmlNodePtr nameNode = xmlFirstElementChild(node);
                            xmlNodePtr priceNode = nameNode->next;
                            xmlNodePtr ratingNode = priceNode->next;

                            if (nameNode != NULL && priceNode != NULL && ratingNode != NULL) {
                                strcpy(products[i].name, (char *)xmlNodeGetContent(nameNode));
                                strcpy(products[i].price, (char *)xmlNodeGetContent(priceNode));
                                strcpy(products[i].rating, (char *)xmlNodeGetContent(ratingNode));
                            }
                        }

                        // Save product information to a CSV file
                        FILE *csvFile = fopen("product_data.csv", "w");
                        if (csvFile != NULL) {
                            fprintf(csvFile, "Product Name, Price, Rating\n");
                            for (i = 0; i < MAX_PRODUCTS; i++) {
                                fprintf(csvFile, "\"%s\", \"%s\", \"%s\"\n", products[i].name, products[i].price, products[i].rating);
                            }
                            fclose(csvFile);
                            printf("Scraping and data storage complete. Data saved to 'product_data.csv'.\n");
                        } else {
                            fprintf(stderr, "Failed to open the CSV file for writing.\n");
                        }
                    }

                    xmlXPathFreeObject(productNodes);
                    xmlXPathFreeContext(context);
                }

                xmlFreeDoc(doc);
            }
        }

        curl_easy_cleanup(curl);
    }

    if (chunk.memory)
        free(chunk.memory);

    return 0;
}
