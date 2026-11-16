
from flask import Flask, jsonify, request

app = Flask(__name__)

cart = []

@app.route("/")
def home():
    return jsonify({
        "service": "Cart Service",
        "status": "Running"
    })

@app.route("/cart", methods=["GET"])
def get_cart():
    return jsonify(cart)

@app.route("/cart", methods=["POST"])
def add_to_cart():
    data = request.get_json()

    if not data or "product_id" not in data:
        return jsonify({"error": "product_id is required"}), 400

    item = {
        "product_id": data["product_id"],
        "quantity": data.get("quantity", 1)
    }

    cart.append(item)
    return jsonify(item), 201

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5002)
