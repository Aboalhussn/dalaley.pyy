from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from werkzeug.security import generate_password_hash, check_password_hash

# تهيئة التطبيق
app = Flask(__name__)

# إعدادات الاتصال بقاعدة البيانات
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dalaley.db'  # استخدام SQLite
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JWT_SECRET_KEY'] = 'your_jwt_secret_key'  # استبدل بـ مفتاح سري قوي

# تهيئة SQLAlchemy و JWT
db = SQLAlchemy(app)
jwt = JWTManager(app)

# نموذج لقاعدة البيانات (العقارات)
class Property(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    description = db.Column(db.String(500), nullable=False)
    price = db.Column(db.Float, nullable=False)
    location = db.Column(db.String(100), nullable=False)
    type = db.Column(db.String(50), nullable=False)  # مثل: شقة، منزل، مكتب

# نموذج لقاعدة البيانات (المستخدمين)
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), nullable=False, unique=True)
    password = db.Column(db.String(200), nullable=False)
    role = db.Column(db.String(50), nullable=False)

# دالة لإنشاء الجداول
@app.before_request
def create_tables():
    db.create_all()

# الصفحة الرئيسية
@app.route('/')
def home():
    return "Welcome to Dalaley Home Page!"

# تسجيل المستخدمين (التسجيل لإنشاء حساب جديد)
@app.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    role = data.get('role')

    # التحقق من أن المستخدم موجود بالفعل
    if User.query.filter_by(username=username).first():
        return jsonify({"message": "User already exists"}), 400

    # تشفير كلمة المرور
    password_hash = generate_password_hash(password)
    new_user = User(username=username, password=password_hash, role=role)
    db.session.add(new_user)
    db.session.commit()

    return jsonify({"message": "User registered successfully"}), 201

# تسجيل الدخول (توليد رمز JWT)
@app.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    # التحقق من المستخدم وكلمة المرور
    user = User.query.filter_by(username=username).first()
    if not user or not check_password_hash(user.password, password):  # التحقق من كلمة المرور المشفرة
        return jsonify({"message": "Invalid credentials"}), 401

    # توليد رمز JWT
    access_token = create_access_token(identity=username)
    return jsonify({"access_token": access_token}), 200

# إضافة عقار جديد
@app.route('/add_property', methods=['POST'])
@jwt_required()
def add_property():
    current_user = get_jwt_identity()  # الحصول على هوية المستخدم الحالي
    data = request.get_json()

    title = data.get('title')
    description = data.get('description')
    price = data.get('price')
    location = data.get('location')
    type = data.get('type')

    # التحقق من أن جميع الحقول موجودة
    if not title or not description or not price or not location or not type:
        return jsonify({"message": "All fields are required"}), 400

    # إضافة العقار إلى قاعدة البيانات
    new_property = Property(title=title, description=description, price=price, location=location, type=type)
    db.session.add(new_property)
    db.session.commit()

    return jsonify({"message": "Property added successfully"}), 201

# الحصول على جميع العقارات
@app.route('/properties', methods=['GET'])
def get_properties():
    properties = Property.query.all()
    results = [{"id": prop.id, "title": prop.title, "description": prop.description, "price": prop.price, "location": prop.location, "type": prop.type} for prop in properties]
    return jsonify(results), 200

# فلترة العقارات حسب السعر أو النوع
@app.route('/filter_properties', methods=['GET'])
def filter_properties():
    price = request.args.get('price', type=float)
    property_type = request.args.get('type', type=str)

    query = Property.query

    # تطبيق الفلاتر
    if price:
        query = query.filter(Property.price <= price)
    if property_type:
        query = query.filter(Property.type == property_type)

    properties = query.all()
    results = [{"id": prop.id, "title": prop.title, "description": prop.description, "price": prop.price, "location": prop.location, "type": prop.type} for prop in properties]
    return jsonify(results), 200

# بدء تشغيل التطبيق
if __name__ == '__main__':
    app.run(debug=True)
