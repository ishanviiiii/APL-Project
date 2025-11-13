# APL-Project
#!/usr/bin/env python3
"""
shopping_cart.py

Single-file implementation of an Online Shopping Cart (CLI) in Python.

Features:
- Product catalog persisted in products.json (auto-creates sample if missing)
- Cart state persisted in cart.json
- Supports PhysicalProduct and DigitalProduct
- Add / remove / update quantity / view cart / checkout operations
- Clean object-oriented structure and input validation
- Meant to be run from terminal:
    python shopping_cart.py
"""

from __future__ import annotations
import json
from abc import ABC, abstractmethod
from dataclasses import dataclass
from pathlib import Path
from typing import Dict, Optional
import sys

# ----------------------------
# Configuration / file paths
# ----------------------------
PRODUCTS_FILE = Path("products.json")
CART_FILE = Path("cart.json")


# ----------------------------
# Domain models
# ----------------------------
class Product(ABC):
    def __init__(self, product_id: str, name: str, price: float, quantity_available: int):
        self._product_id = str(product_id)
        self._name = str(name)
        self._price = float(price)
        self._quantity_available = int(quantity_available)

    @property
    def product_id(self) -> str:
        return self._product_id

    @property
    def name(self) -> str:
        return self._name

    @property
    def price(self) -> float:
        return self._price

    @property
    def quantity_available(self) -> int:
        return self._quantity_available

    @quantity_available.setter
    def quantity_available(self, v: int):
        if v < 0:
            raise ValueError("Quantity cannot be negative")
        self._quantity_available = int(v)

    def decrease_quantity(self, amount: int) -> bool:
        if amount <= 0:
            return False
        if self._quantity_available >= amount:
            self._quantity_available -= amount
            return True
        return False

    def increase_quantity(self, amount: int) -> None:
        if amount > 0:
            self._quantity_available += amount

    @abstractmethod
    def display_details(self) -> str:
        ...

    @abstractmethod
    def to_dict(self) -> dict:
        ...


class PhysicalProduct(Product):
    def __init__(self, product_id: str, name: str, price: float, quantity_available: int, weight: float = 0.0):
        super().__init__(product_id, name, price, quantity_available)
        self._weight = float(weight)

    @property
    def weight(self) -> float:
        return self._weight

    def display_details(self) -> str:
        return f"[{self.product_id}] {self.name} — ₹{self.price:.2f} | Weight: {self.weight}kg | Stock: {self.quantity_available}"

    def to_dict(self) -> dict:
        return {
            "type": "physical",
            "product_id": self.product_id,
            "name": self.name,
            "price": self.price,
            "quantity_available": self.quantity_available,
            "weight": self.weight
        }


class DigitalProduct(Product):
    def __init__(self, product_id: str, name: str, price: float, quantity_available: int, download_link: str = ""):
        super().__init__(product_id, name, price, quantity_available)
        self._download_link = str(download_link)

    @property
    def download_link(self) -> str:
        return self._download_link

    def display_details(self) -> str:
        return f"[{self.product_id}] {self.name} — ₹{self.price:.2f} | Download: {self.download_link} | Licenses: {self.quantity_available}"

    def to_dict(self) -> dict:
        return {
            "type": "digital",
            "product_id": self.product_id,
            "name": self.name,
            "price": self.price,
            "quantity_available": self.quantity_available,
            "download_link": self.download_link
        }


# ----------------------------
# Cart item
# ----------------------------
@dataclass
class CartItem:
    product: Product
    quantity: int

    def subtotal(self) -> float:
        return self.product.price * self.quantity

    def to_dict(self) -> dict:
        return {"product_id": self.product.product_id, "quantity": self.quantity}


# ----------------------------
# ShoppingCart manager
# ----------------------------
class ShoppingCart:
    def __init__(self, products_file: Path = PRODUCTS_FILE, cart_file: Path = CART_FILE):
        self.products_file = products_file
        self.cart_file = cart_file
        self.catalog: Dict[str, Product] = self._load_catalog()
        self.items: Dict[str, CartItem] = {}
        self._load_cart_state()

    # -------------------------
    # Catalog persistence
    # -------------------------
    def _create_sample_products(self) -> None:
        sample = [
            {"type": "physical", "product_id": "P001", "name": "WIFI Router", "price": 599.00, "quantity_available": 72, "weight": 0.5},
            {"type": "physical", "product_id": "P002", "name": "Keyboard", "price": 550.00, "quantity_available": 81, "weight": 1.1},
            {"type": "digital",  "product_id": "D001", "name": "E-book: Skill The Python", "price": 99.99, "quantity_available": 999, "download_link": "https://example.com/python-ebook"},
            {"type": "physical", "product_id": "P003", "name": "CPU", "price": 1000.00, "quantity_available": 75, "weight": 5.0},
            {"type": "physical", "product_id": "P004", "name": "Smart-TV", "price": 13000.00, "quantity_available": 55, "weight": 4.5}
        ]
        with self.products_file.open("w", encoding="utf-8") as f:
            json.dump(sample, f, indent=2, ensure_ascii=False)

    def _load_catalog(self) -> Dict[str, Product]:
        if not self.products_file.exists():
            # create sample and load
            self._create_sample_products()

        try:
            with self.products_file.open("r", encoding="utf-8") as f:
                data = json.load(f)
        except Exception:
            return {}

        catalog: Dict[str, Product] = {}
        for p in data:
            ptype = p.get("type", "physical")
            pid = str(p["product_id"])
            try:
                if ptype == "physical":
                    prod = PhysicalProduct(pid, p["name"], float(p["price"]), int(p["quantity_available"]), float(p.get("weight", 0.0)))
                elif ptype == "digital":
                    prod = DigitalProduct(pid, p["name"], float(p["price"]), int(p["quantity_available"]), p.get("download_link", ""))
                else:
                    prod = PhysicalProduct(pid, p["name"], float(p["price"]), int(p["quantity_available"]), float(p.get("weight", 0.0)))
                catalog[pid] = prod
            except Exception as e:
                # skip malformed entries
                print(f"Warning: skipping product {pid} due to error: {e}")
        return catalog

    def _save_catalog(self) -> None:
        arr = [p.to_dict() for p in self.catalog.values()]
        with self.products_file.open("w", encoding="utf-8") as f:
            json.dump(arr, f, indent=2, ensure_ascii=False)

    # -------------------------
    # Cart persistence
    # -------------------------
    def _load_cart_state(self) -> None:
        if not self.cart_file.exists():
            return
        try:
            with self.cart_file.open("r", encoding="utf-8") as f:
                data = json.load(f)
        except Exception:
            return
        for item in data:
            pid = item.get("product_id")
            qty = int(item.get("quantity", 0))
            prod = self.catalog.get(pid)
            if prod and qty > 0:
                self.items[pid] = CartItem(prod, qty)

    def _save_cart_state(self) -> None:
        arr = [ci.to_dict() for ci in self.items.values()]
        with self.cart_file.open("w", encoding="utf-8") as f:
            json.dump(arr, f, indent=2, ensure_ascii=False)

    # -------------------------
    # Cart operations
    # -------------------------
    def list_products(self) -> None:
        if not self.catalog:
            print("No products available.")
            return
        print("\nAvailable Products:")
        print("-------------------")
        for p in self.catalog.values():
            print(p.display_details())

    def view_product(self, product_id: str) -> None:
        p = self.catalog.get(product_id)
        if not p:
            print("Product not found.")
            return
        print(p.display_details())

    def add_item(self, product_id: str, quantity: int) -> bool:
        if quantity <= 0:
            print("Quantity must be positive.")
            return False
        product = self.catalog.get(product_id)
        if not product:
            print("Product does not exist.")
            return False
        if product.quantity_available < quantity:
            print(f"Insufficient stock. Available: {product.quantity_available}")
            return False
        if product_id in self.items:
            self.items[product_id].quantity += quantity
        else:
            self.items[product_id] = CartItem(product, quantity)
        product.decrease_quantity(quantity)
        self._save_cart_state()
        self._save_catalog()
        print(f"Added {quantity} x {product.name} to cart.")
        return True

    def remove_item(self, product_id: str) -> bool:
        item = self.items.get(product_id)
        if not item:
            print("Item not found in cart.")
            return False
        # return quantity to stock
        item.product.increase_quantity(item.quantity)
        del self.items[product_id]
        self._save_cart_state()
        self._save_catalog()
        print(f"Removed {item.product.name} from cart.")
        return True

    def update_quantity(self, product_id: str, new_quantity: int) -> bool:
        if new_quantity < 0:
            print("Quantity cannot be negative.")
            return False
        item = self.items.get(product_id)
        if not item:
            print("Item not in cart.")
            return False
        current = item.quantity
        diff = new_quantity - current
        if diff == 0:
            print("Quantity unchanged.")
            return True
        if diff > 0:
            # need to reserve additional stock
            if item.product.quantity_available < diff:
                print(f"Insufficient stock to increase. Available: {item.product.quantity_available}")
                return False
            item.product.decrease_quantity(diff)
            item.quantity = new_quantity
        else:
            # returning stock
            item.product.increase_quantity(-diff)
            if new_quantity == 0:
                del self.items[product_id]
            else:
                item.quantity = new_quantity
        self._save_cart_state()
        self._save_catalog()
        print("Cart updated.")
        return True

    def get_total(self) -> float:
        return sum(ci.subtotal() for ci in self.items.values())

    def display_cart(self) -> None:
        if not self.items:
            print("\nYour cart is empty.")
            return
        print("\nYour Cart:")
        print("----------")
        for ci in self.items.values():
            print(f"{ci.product.name} (ID: {ci.product.product_id}) | Qty: {ci.quantity} | Unit: ₹{ci.product.price:.2f} | Subtotal: ₹{ci.subtotal():.2f}")
        print("----------")
        print(f"Total Amount: ₹{self.get_total():.2f}")

    def checkout(self) -> None:
        if not self.items:
            print("Cart is empty. Nothing to checkout.")
            return
        self.display_cart()
        confirm = input("Proceed to checkout? (y/n): ").strip().lower()
        if confirm != "y":
            print("Checkout canceled.")
            return
        # In a real system, payment and order creation would occur here
        total = self.get_total()
        # clear cart (no stock return, because we already deducted on add)
        self.items.clear()
        # persist emptied cart
        self._save_cart_state()
        print("\nCheckout successful!")
        print(f"Amount paid: ₹{total:.2f}")
        print("Thank you for shopping with us!")

    # Utility: add new product (admin-like)
    def add_product_to_catalog(self, type_: str, product_id: str, name: str, price: float, qty: int, extra: Optional[str] = None) -> None:
        if product_id in self.catalog:
            print("Product ID already exists.")
            return
        if type_ == "physical":
            weight = float(extra) if extra else 0.0
            prod = PhysicalProduct(product_id, name, price, qty, weight)
        else:
            link = str(extra) if extra else ""
            prod = DigitalProduct(product_id, name, price, qty, link)
        self.catalog[product_id] = prod
        self._save_catalog()
        print(f"Product {name} added to catalog.")


# ----------------------------
# CLI / main
# ----------------------------
def clear_screen():
    # simple newline-based clear to keep compatibility in all terminals
    print("\n" * 3)


def main():
    cart = ShoppingCart()

    MENU = """
==== Online Shopping Cart ====
1. View Products
2. View Product Details
3. Add Item to Cart
4. View Cart
5. Update Item Quantity in Cart
6. Remove Item from Cart
7. Checkout
8. Admin: Add Product to Catalog
9. Exit
================================
"""

    while True:
        print(MENU)
        choice = input("Enter your choice (1-9): ").strip()
        if choice == "1":
            cart.list_products()
        elif choice == "2":
            pid = input("Enter Product ID: ").strip()
            cart.view_product(pid)
        elif choice == "3":
            pid = input("Enter Product ID to add: ").strip()
            try:
                qty = int(input("Enter quantity: ").strip())
            except ValueError:
                print("Invalid quantity.")
                continue
            cart.add_item(pid, qty)
        elif choice == "4":
            cart.display_cart()
        elif choice == "5":
            pid = input("Enter Product ID in cart to update: ").strip()
            try:
                new_q = int(input("Enter new quantity (0 to remove): ").strip())
            except ValueError:
                print("Invalid quantity.")
                continue
            cart.update_quantity(pid, new_q)
        elif choice == "6":
            pid = input("Enter Product ID to remove from cart: ").strip()
            cart.remove_item(pid)
        elif choice == "7":
            cart.checkout()
        elif choice == "8":
            # admin add product
            t = input("Type (physical/digital): ").strip().lower()
            pid = input("Product ID: ").strip()
            name = input("Name: ").strip()
            try:
                price = float(input("Price: ").strip())
                qty = int(input("Initial quantity: ").strip())
            except ValueError:
                print("Invalid numeric input.")
                continue
            extra = ""
            if t == "physical":
                extra = input("Weight (kg): ").strip()
            else:
                extra = input("Download link (optional): ").strip()
            cart.add_product_to_catalog(t, pid, name, price, qty, extra)
        elif choice == "9":
            print("Exiting. Goodbye!")
            sys.exit(0)
        else:
            print("Invalid choice. Please choose a valid option.")

        input("\nPress Enter to continue...")
        clear_screen()


if __name__ == "__main__":
    main()
