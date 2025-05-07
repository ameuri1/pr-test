# Import necessary libraries
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager, create_access_token
from werkzeug.security import generate_password_hash, check_password_hash
from sqlalchemy.exc import IntegrityError


# Initialize Flask app
app = Flask(__name__)

# Configure the database and JWT secret key
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///neighborhood.db"  # SQLite database for local development
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False  # Disable unnecessary warnings
app.config["JWT_SECRET_KEY"] = "your_secret_key_here"  # Replace with a secure key

# Initialize database and JWT manager
db = SQLAlchemy(app)
jwt = JWTManager(app)


# Define the User model for authentication
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)  # Unique ID for each user
    username = db.Column(db.String(50), unique=True, nullable=False)  # User's unique username
    password = db.Column(db.String(100), nullable=False)  # Hashed password


# API route to register a new user
@app.route("/register", methods=["POST"])
def register():
    data = request.get_json()
    if not data.get("username") or not data.get("password"):
        return jsonify({"error": "Username and password are required"}), 400

    hashed_password = generate_password_hash(data["password"], method="sha256")  # Hash the password
    new_user = User(username=data["username"], password=hashed_password)
    try:
        db.session.add(new_user)
        db.session.commit()
        return jsonify({"message": "User registered successfully"}), 201
    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "Username already exists"}), 409


# API route to log in a user
@app.route("/log_in", methods=["POST"])
def log_in():
    data = request.get_json()
    user = User.query.filter_by(username=data["username"]).first()
    if not user or not check_password_hash(user.password, data["password"]):
        return jsonify({"error": "Invalid credentials"}), 401

    access_token = create_access_token(identity={"username": user.username})
    return jsonify({"access_token": access_token})


# Define the Alert model
class Alert(db.Model):
    id = db.Column(db.Integer, primary_key=True)  # Unique ID for each alert
    title = db.Column(db.String(100), nullable=False)  # Alert title
    description = db.Column(db.String(500), nullable=False)  # Alert description


# API route to create a new alert
@app.route("/alerts", methods=["POST"])
def create_alert():
    data = request.get_json()
    new_alert = Alert(title=data["title"], description=data["description"])
    db.session.add(new_alert)
    db.session.commit()
    return jsonify({"message": "Alert created successfully"}), 201


# API route to get all alerts
@app.route("/alerts", methods=["GET"])
def get_alerts():
    alerts = Alert.query.all()
    alert_list = [
        {
            "id": alert.id,
            "title": alert.title,
            "description": alert.description,
        }
        for alert in alerts
    ]
    return jsonify(alert_list), 200


# Define the Event model
class Event(db.Model):
    id = db.Column(db.Integer, primary_key=True)  # Unique ID for each event
    name = db.Column(db.String(100), nullable=False)  # Event name
    location = db.Column(db.String(200), nullable=False)  # Event location
    date = db.Column(db.String(50), nullable=False)  # Event date


# API route to create a new event
@app.route("/events", methods=["POST"])
def create_event():
    data = request.get_json()
    new_event = Event(name=data["name"], location=data["location"], date=data["date"])
    db.session.add(new_event)
    db.session.commit()
    return jsonify({"message": "Event created successfully"}), 201


# API route to get all events
@app.route("/events", methods=["GET"])
def get_events():
    events = Event.query.all()
    event_list = [
        {
            "id": event.id,
            "name": event.name,
            "location": event.location,
            "date": event.date,
        }
        for event in events
    ]
    return jsonify(event_list), 200


# Define the Report model
class Report(db.Model):
    id = db.Column(db.Integer, primary_key=True)  # Unique ID for each report
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)  # ID of the user who submitted the report
    content = db.Column(db.String(500), nullable=False)  # Report content


# API route to create a new report
@app.route("/reports", methods=["POST"])
def create_report():
    data = request.get_json()
    new_report = Report(user_id=data["user_id"], content=data["content"])
    db.session.add(new_report)
    db.session.commit()
    return jsonify({"message": "Report submitted successfully"}), 201


# API route to get all reports
@app.route("/reports", methods=["GET"])
def get_reports():
    reports = Report.query.all()
    report_list = [
        {
            "id": report.id,
            "user_id": report.user_id,
            "content": report.content,
        }
        for report in reports
    ]
    return jsonify(report_list), 200


# Run the application and initialize the database
if __name__ == "__main__":
    db.create_all()  # Create database tables based on models
    app.run(debug=True)  # Enable debug mode for development

# One blank line here
