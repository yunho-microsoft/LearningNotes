# Design book selling platform

Ref: https://www.hellointerview.com/community/questions/book-seller-platform/cm8j5hkhi009ouxg8w34ni7nd
Sell register its name and book inventories

Questions:
- Does payment happen in the platform? Or payment happens on the seller's site?
- How do we get book listing information from sellers?
    - Do they register proactively using the register API?
    - Do we need to crawl data from their websites?
        If so, need crawler and design is much out of the scope of an interview
    - Or each seller has their API to get book inventory?
        If so, we need a job to get the data
- How does user search for a book? Or user directly send request for a bid?
    Probably need to assume user directly send bids with book name and price, since search is also a big topic

# Register sellor
PUT /sellers/register
{
    name: str,
    website_link: str,
}

# Update inventory
PUT or PATCH /sellers/{seller_id}/books
{
    inventory: [
        books: {
            "name": str,
            "price": number,
            "inventory": number,
        }
    ]
}

# Create a bid
POST /orders/
{
    name: str
    max_price: number
    payment_info: {
        ...
    }
}

return order_id

# GET /orders/{order_id}

Query listings
GET /book/{book_id}?sort=default&limit10


# Other requirements:
QPS 10k per sec
listing is


# Designs
## DB
Choose SQL DB since data is structured.
Tables:
- Sellers: seller information
    - id: PK
    - name: indexed
    - website links

- Books: book information
    - id: PK
    - name: indexed
    - release_date

- Inventory
    - id: PK
    - book_id: Foreign Key (FK), indexed
    - sellor_id: FK, indexed
    - price
    - count

- Users
    - id
    - name
    - email: indexed

- Orders
    - id
    - user_id
    - book_id: FK
    - bid_price
    - price: price that the order is completed
    - date
    - status

### Data access pattern
- Register new seller
    - update sellers table

- Update inventory by book name
    - Assume we call each seller's API to update inventory, we already have the seller_id
    - Need to JOIN Inventory and Books table

- Query prices of a book using its name
    - Needs to join Inventory and Books table.
        - Maybe we can denormalize by adding book_name to the Inventory table since book name usually should not be changed

- Create a new order(bid)
    - Update Orders table
    - Get lowest price
        - Query Inventory table with book_name

## Service:

Seller request --> API gateway --> SellerManager --> DB
User request   -|                  OrderManager --> QueueService --> OrderWorker
                                   InventorySearch <-- DB

SellerManager: manage sellor information and inventory
OrderManager: manage client's order
InventorySearch: Search for best seller from price
