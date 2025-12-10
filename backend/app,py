import os
import sys
import base64
from flask import Flask, request, jsonify
from flask_cors import CORS
from pymongo import MongoClient, ReturnDocument
from bson import ObjectId
from dotenv import load_dotenv

# --- CONFIGURATION ---
load_dotenv()

app = Flask(__name__)
CORS(app)

# 1. VERIFY ENVIRONMENT VARIABLES
mongo_uri = os.getenv("MONGO_URI")
if not mongo_uri:
    print("❌ ERROR: 'MONGO_URI' is missing from .env file!")
    sys.exit(1)

# 2. MONGODB CONNECTION
try:
    client = MongoClient(mongo_uri)
    client.admin.command('ping')
    
    db = client['flipr_assignment_db']
    
    # Collections
    projects_collection = db['projects']
    clients_collection = db['clients']
    contacts_collection = db['contacts']
    subscribers_collection = db['subscribers']
    counter_collection = db['counters']  # For auto-increment IDs
    
    print("✅ Connected to MongoDB Atlas Successfully!")

except Exception as e:
    print(f"❌ CRITICAL DATABASE ERROR: {e}")
    sys.exit(1)

# Helper function to get next auto-increment ID
def get_next_id(collection_name):
    # Atomically increment and return the new sequence value
    counter = counter_collection.find_one_and_update(
        {'_id': collection_name},
        {'$inc': {'sequence_value': 1}},
        return_document=ReturnDocument.AFTER,
        upsert=True
    )
    if not counter:
        # initialize if somehow missing
        counter_collection.update_one({'_id': collection_name}, {'$set': {'sequence_value': 1}}, upsert=True)
        return 1
    return int(counter.get('sequence_value', 1))

@app.route('/')
def home():
    return "Backend is running (Images stored in DB)!"

# ==========================================
# PUBLIC ROUTES
# ==========================================

@app.route('/api/projects', methods=['GET'])
def get_projects():
    projects = list(projects_collection.find({}, {'_id': 0}))
    return jsonify({"projects": projects})

@app.route('/api/clients', methods=['GET'])
def get_clients():
    clients = list(clients_collection.find({}, {'_id': 0}))
    return jsonify({"clients": clients})

@app.route('/api/contact', methods=['POST'])
def submit_contact():
    data = request.json
    if not data:
        return jsonify({"error": "No data provided"}), 400
    contacts_collection.insert_one(data)
    return jsonify({"message": "Contact submitted"}), 201

@app.route('/api/subscribe', methods=['POST'])
def subscribe():
    data = request.json
    if not data or 'email' not in data:
        return jsonify({"error": "Email is required"}), 400
    subscribers_collection.insert_one(data)
    return jsonify({"message": "Subscribed"}), 201

# ==========================================
# ADMIN ROUTES (Base64 Image Storage)
# ==========================================

@app.route('/api/admin/project', methods=['POST'])
def add_project():
    try:
        if 'image' not in request.files:
            return jsonify({"error": "No image uploaded"}), 400
        
        file = request.files['image']
        name = request.form.get('name')
        description = request.form.get('description')

        # VALIDATE REQUIRED FIELDS
        if not name or not description:
            return jsonify({"error": "Name and description are required"}), 400

        # CONVERT IMAGE TO BASE64 STRING
        if file:
            encoded_string = base64.b64encode(file.read()).decode('utf-8')
            # Create the data URL (e.g., "data:image/jpeg;base64,sd23...")
            image_url = f"data:{file.content_type};base64,{encoded_string}"

            # Generate auto-increment ID
            project_id = get_next_id('projects')
            
            project_data = {
                "id": project_id,
                "name": name,
                "description": description,
                "image_url": image_url
            }
            result = projects_collection.insert_one(project_data)
            # include inserted id as string to avoid ObjectId serialization errors
            project_data["_id"] = str(result.inserted_id)
            return jsonify({"message": "Project created", "data": project_data}), 201

    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/api/admin/client', methods=['POST'])
def add_client():
    try:
        if 'image' not in request.files:
            return jsonify({"error": "No image uploaded"}), 400
        
        file = request.files['image']
        name = request.form.get('name')
        description = request.form.get('description')
        designation = request.form.get('designation')

        # VALIDATE REQUIRED FIELDS
        if not name or not description or not designation:
            return jsonify({"error": "Name, description, and designation are required"}), 400

        # CONVERT IMAGE TO BASE64 STRING
        if file:
            encoded_string = base64.b64encode(file.read()).decode('utf-8')
            image_url = f"data:{file.content_type};base64,{encoded_string}"

            # Generate auto-increment ID
            client_id = get_next_id('clients')
            
            client_data = {
                "id": client_id,
                "name": name,
                "description": description,
                "designation": designation,
                "image_url": image_url
            }
            result = clients_collection.insert_one(client_data)
            client_data["_id"] = str(result.inserted_id)
            return jsonify({"message": "Client added", "data": client_data}), 201

    except Exception as e:
        print(e)
        return jsonify({"error": str(e)}), 500

@app.route('/api/admin/contacts', methods=['GET'])
def get_all_contacts():
    contacts = list(contacts_collection.find({}, {'_id': 0}))
    return jsonify({"contacts": contacts})

@app.route('/api/admin/subscribers', methods=['GET'])
def get_all_subscribers():
    subscribers = list(subscribers_collection.find({}, {'_id': 0}))
    return jsonify({"subscribers": subscribers})

if __name__ == '__main__':
    app.run(debug=True, port=5000)