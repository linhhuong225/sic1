# sic1
import requests
import json
import time
from tqdm import tqdm

# Token dùng chung (nên lưu trong 1 biến)
AUTH_TOKEN = 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....'

HEADERS = {
    'authorization': AUTH_TOKEN,
    'content-type': 'application/json',
}

def call_api(url, method="POST", payload=None, headers=HEADERS, retries=3):
    for attempt in range(retries):
        try:
            if method.upper() == "POST":
                response = requests.post(url, headers=headers, json=payload)
            else:
                response = requests.get(url, headers=headers, params=payload)
            response.raise_for_status()
            return response.json()
        except Exception as e:
            print(f"Error on {url}: {e}")
            time.sleep(2)
    return None

def get_home_collections(lat, long):
    url = "https://gw.be.com.vn/api/v1/be-merchant-gateway/web/customer/get_home_collections"
    payload = {
        "locale": "vi",
        "client_info": {"locale": "vi", "device_type": 3},
        "latitude": lat,
        "longitude": long
    }
    return call_api(url, payload=payload)

def get_restaurants(collection_id, lat, long, limit=10, page=1):
    url = "https://gw.be.com.vn/api/v1/be-marketplace/web/collection/items/restaurants"
    payload = {
        "collection_id": str(collection_id),
        "page": page,
        "limit": limit,
        "locale": "vi",
        "latitude": lat,
        "longitude": long
    }
    return call_api(url, payload=payload)

def get_restaurant_detail(restaurant_id, lat, long):
    url = "https://gw.be.com.vn/api/v1/be-marketplace/web/restaurant/detail"
    payload = {
        "restaurant_id": str(restaurant_id),
        "locale": "vi",
        "latitude": lat,
        "longitude": long
    }
    return call_api(url, payload=payload)

def main(lat=10.7725, long=106.6980):
    data = get_home_collections(lat, long)
    if not data:
        print("Không lấy được nhóm ngành.")
        return

    for i, collection in tqdm(enumerate(data.get('collections', [])), total=len(data.get('collections', []))):
        print(f"Nhóm ngành {i+1}: {collection['name']} (ID: {collection['id']})")

        restaurant_data = get_restaurants(collection_id=collection['id'], lat=lat, long=long)
        if not restaurant_data:
            continue

        for restaurant in restaurant_data.get('data', []):
            try:
                print(f" - Nhà hàng: {restaurant['name']} (ID: {restaurant['restaurant_id']})")
                detail = get_restaurant_detail(restaurant_id=restaurant['restaurant_id'], lat=lat, long=long)
                if detail and 'data' in detail:
                    for category in detail['data'].get('categories', []):
                        print(f"  - Danh mục: {category['category_name']}")
                        for item in category.get('items', []):
                            print(f"    + Món: {item['item_name']} - Giá: {item['display_price']}")
                        break
                time.sleep(1)  # Sleep để tránh spam server
            except Exception as e:
                print(f"Error processing restaurant {restaurant['restaurant_id']}: {e}")
            break  # Chỉ lấy 1 nhà hàng demo, bỏ dòng này nếu muốn lấy tất cả

if __name__ == "__main__":
    main()
