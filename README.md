# Hospital-management-system-
html CSS JavaScript Python 
from flask import Flask, render_template, session, request, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
import mysql.connector
from datetime import datetime
import json
from flask_mail import Mail


with open("config.json", "r") as c:
    params = json.load(c)["params"]

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = params["local_server_uri"]
connection = mysql.connector.connect(host=params['db_host'], database=params['db_database'], user=params['db_user'])
cursor = connection.cursor()
db = SQLAlchemy(app)
app.config.update(
    MAIL_SERVER='smtp.gmail.com',
    MAIL_PORT='465',
    MAIL_USE_SSL=True,
    MAIL_USERNAME=params["gmail_username"],
    MAIL_PASSWORD=params["gmail_password"]
)
mail = Mail(app)
app.config['SECRET_KEY'] = params['secret_key']


class Patient(db.Model):
    Patient_no = db.Column(db.Integer, primary_key=True)
    Patient_name = db.Column(db.String(50), nullable=False)
    Patient_address = db.Column(db.String(100), nullable=True)
    Age = db.Column(db.String(3), nullable=True)
    Sex = db.Column(db.String(1), nullable=True)
    Reffered_Department = db.Column(db.String(50), nullable=True)
    Reffered_Doctor = db.Column(db.String(50), nullable=True)
    Prescription = db.Column(db.String(1000), nullable=True)
    Doctor_charges = db.Column(db.String(10), nullable=True)
    Doctor_Charges_Paid = db.Column(db.String(8), nullable=True)
    Reg_Date = db.Column(db.String(20), nullable=True)
    Admit_in_hospital = db.Column(db.String(5), nullable=True)
    Room = db.Column(db.String(11), nullable=True)
    Room_Status = db.Column(db.String(8), nullable=True)
    Room_allocation_fee = db.Column(db.String(7), nullable=True)
    Room_fee_status = db.Column(db.String(8), nullable=True)
    Room_fee_status = db.Column(db.String(8), nullable=True)
    Discharge = db.Column(db.String(5), nullable=True)


class PatientDischarge(db.Model):
    patient_no = db.Column(db.Integer, primary_key=True)
    department = db.Column(db.String(50), nullable=False)
    room_no = db.Column(db.String(50), nullable=False)
    room_charges = db.Column(db.Integer, nullable=False)
    test_charges = db.Column(db.Integer, nullable=False)
    operation_charges = db.Column(db.Integer, unique=True, nullable=True)
    blood_charges = db.Column(db.Integer, nullable=True)
    doctor_charges = db.Column(db.Integer, nullable=True)
    total_charges = db.Column(db.Integer, nullable=True)
    payment_status = db.Column(db.String(10), nullable=True)
    discharge_date = db.Column(db.String(50), nullable=True)


class AllDoctors(db.Model):
    Doc_id = db.Column(db.Integer, primary_key=True)
    Doc_name = db.Column(db.String(50), nullable=False)
    Doc_rank = db.Column(db.String(25), nullable=False)
    Doc_phone = db.Column(db.String(15), unique=True, nullable=False)
    Department = db.Column(db.String(50), nullable=True)


class DoctorPrescription(db.Model):
    serial_no = db.Column(db.Integer, primary_key=True)
    Patient_no = db.Column(db.String(11), nullable=False)
    Prescription = db.Column(db.String(1000), nullable=False)


class PatientContacts(db.Model):
        serial_no = db.Column(db.Integer, primary_key=True)
        city = db.Column(db.String(25), nullable=False)
        country = db.Column(db.String(25), nullable=False)
        email = db.Column(db.String(100), nullable=False)
        phone = db.Column(db.String(100), nullable=False)


@app.route("/", methods=["GET", "POST"])
def login():
    if request.method == 'POST':
        user_email = request.form.get("email")
        user_password = request.form.get("password")
        cursor.execute('SELECT * FROM Login where email=%s and password=%s', (user_email, user_password))
        record = cursor.fetchone()
        if record:
            session['logged_in'] = True
            session['username'] = record[1]
            return redirect(url_for('p_registration'))

    return render_template("login.html", params=params)


@app.route("/patient-registration", methods=["GET", "POST"])
def p_registration():
    cursor.execute('SELECT department_name FROM department')
    departments = cursor.fetchall()
    cursor.execute('SELECT Doc_name, Department FROM all_doctors ORDER BY Department ASC;')
    doc_names = cursor.fetchall()
    if request.method == "POST":
        token = request.form.get('patient-no')
        p_first_name = request.form.get('first_name')
        p_last_name = request.form.get('last_name')
        p_f_city = request.form.get('city')
        p_f_country = request.form.get('country')
        p_f_email = request.form.get('email')
        p_f_phone = request.form.get('phone')
        p_f_address = request.form.get('address')
        p_f_age = request.form.get('age')
        p_f_sex = request.form.get('gender')
        p_f_doc = request.form.get('reffer-doc')
        p_f_dept = request.form.get('reffer-depart')
        p_f_d_fee = request.form.get('doctor_fee')
        p_f_p_status = request.form.get('payment_status')
        entry = Patient(Patient_no=token, Patient_name=p_first_name+" "+p_last_name,
                        Patient_address=p_f_address, Age=p_f_age, Sex=p_f_sex, Reffered_Department=p_f_dept,
                        Reffered_Doctor=p_f_doc, Doctor_charges=p_f_d_fee, Doctor_Charges_Paid=p_f_p_status,
                        Reg_Date=datetime.now())
        db.session.add(entry)
        db.session.commit()
        entry1 = PatientContacts(city=p_f_city, country=p_f_country, email=p_f_email, phone=p_f_phone)
        db.session.add(entry1)
        db.session.commit()
        mail.send_message(
            'The University of Lahore Teaching Hospital: Patient Registration',
            sender=params['gmail_username'],
            recipients=[p_f_email],
            body=f'Thank you for choosing The University of Lahore Teaching Hospital\n'
                 f'Patient No: {token}\nPatient Name: {p_first_name+" "+p_last_name}\nCity: {p_f_city}\n'
                 f'Country: {p_f_country}\nEmail: {p_f_email}\nAddress: {p_f_address}\nAge: {p_f_age}\n'
                 f'Sex: {p_f_sex}\nReffered Department: {p_f_dept}\nReffered Doctor: {p_f_doc}\n'
                 f'Doctor Charges: {p_f_d_fee}\nDoctor Charges Paid: {p_f_p_status}'
                 f'\nRegistration Date: {datetime.now()}'
        )
        return render_template("p_registration.html", params=params, departments=departments, doc_names=doc_names,
                               token=token)

    return render_template("p_registration.html", params=params, departments=departments, doc_names=doc_names)


@app.route("/doctor-registration", methods=["GET", "POST"])
def d_registration():
    cursor.execute('SELECT department_name FROM department')
    departments = cursor.fetchall()
    if request.method == "POST":
        d_prefix = request.form.get('prefix')
        d_name = request.form.get('Doc_name')
        d_rank = request.form.get('rank')
        d_phone = request.form.get('phone')
        d_dept = request.form.get('department')
        entry = AllDoctors(Doc_name=d_prefix+d_name, Doc_rank=d_rank,
                           Doc_phone=d_phone, Department=d_dept)
        db.session.add(entry)
        db.session.commit()
        return render_template("doctor_registration.html", params=params, departments=departments)

    return render_template("doctor_registration.html", params=params, departments=departments)


@app.route("/room-allocation", methods=["GET", "POST"])
def r_allocation():
    cursor.execute('SELECT department_name FROM department')
    departments = cursor.fetchall()
    cursor.execute('SELECT room_no from room WHERE status = "Y";')
    rooms = cursor.fetchall()
    if request.method == "POST":
        patient_no = request.form.get('patient-no')
        room_no = request.form.get('room')
        room_status = request.form.get('room_status')
        room_fee = request.form.get('room_fee')
        payment_status = request.form.get('payment_status')
        cursor.execute(f'UPDATE `patient` SET `Admit_in_hospital` = "Yes" WHERE'
                       f' `patient`.`Patient_no` = "{patient_no}";')
        cursor.execute(f'UPDATE `patient` SET `Room` = "{room_no}" WHERE `patient`.`Patient_no` = "{patient_no}";')
        cursor.execute(f'UPDATE `patient` SET `Room_Status` = "{room_status}" WHERE'
                       f' `patient`.`Patient_no` = "{patient_no}";')
        cursor.execute(f'UPDATE `patient` SET `Room_allocation_fee` = {room_fee} WHERE'
                       f' `patient`.`Patient_no` = "{patient_no}";')
        cursor.execute(f'UPDATE `patient` SET `Room_fee_status` = "{payment_status}" WHERE'
                       f' `patient`.`Patient_no` = "{patient_no}";')
        cursor.execute(f'UPDATE `room` SET `status` = "N" WHERE `room`.`room_no` = "{room_no}";')
        connection.commit()
        cursor.execute(f'SELECT Email FROM patient WHERE Patient_no = "{patient_no}";')
        email = cursor.fetchone()
        cursor.execute(f'SELECT Patient_name FROM patient WHERE Patient_no = "{patient_no}";')
        name = cursor.fetchone()
        mail.send_message(
            'The University of Lahore Teaching Hospital: Patient Discharge',
            sender=params['gmail_username'],
            recipients=[email[0]],
            body=f'Thank you for choosing The University of Lahore Teaching Hospital\n'
                 f'Patient No: {patient_no}\nPatient Name: {name[0]}\nRoom No: {room_no}\nRoom Status:{room_status}\n'
                 f'Room Fee:{room_fee}\nPayment Status:{payment_status}'
        )
        return render_template("room_allocation.html", params=params, departments=departments, rooms=rooms)

    return render_template("room_allocation.html", params=params, departments=departments, rooms=rooms)


@app.route("/doctor-records")
def doctor_records():
    doctors = AllDoctors.query.all()
    return render_template("doctor_records.html", params=params, doctors=doctors)


@app.route("/patient-records")
def patient_records():
    patients = Patient.query.all()
    return render_template("patient_records.html", params=params, patients=patients)


@app.route("/patient-discharge", methods=["GET", "POST"])
def p_discharge():
    cursor.execute('SELECT department_name FROM department')
    departments = cursor.fetchall()
    cursor.execute('SELECT room_no from room WHERE status = "N";')
    rooms = cursor.fetchall()
    if request.method == 'POST':
        patient_no = request.form.get('patient-no')
        department = request.form.get('department')
        room = request.form.get('room')
        room_charges = request.form.get('room_charges')
        test_charges = request.form.get('test_charges')
        operation_charges = request.form.get('operation_charges')
        blood_charges = request.form.get('blood_charges')
        doctor_charges = request.form.get('doctor_charges')
        total_charges = request.form.get('total_charges')
        payment_status = request.form.get('payment_status')
        entry = PatientDischarge(patient_no=patient_no, department=department, room_no=room,
                                 room_charges=room_charges, test_charges=test_charges,
                                 operation_charges=operation_charges, blood_charges=blood_charges,
                                 doctor_charges=doctor_charges,
                                 total_charges=total_charges, payment_status=payment_status,
                                 discharge_date=datetime.now())
        db.session.add(entry)
        db.session.commit()
        cursor.execute(f'UPDATE `room` SET `status` = "Y" WHERE `room`.`room_no` = "{room}";')
        cursor.execute(f'UPDATE `patient` SET `Discharge` = "Yes" WHERE `patient`.`Patient_no` = "{patient_no}";')
        connection.commit()
        cursor.execute(f'SELECT Email FROM patient WHERE Patient_no = "{patient_no}";')
        email = cursor.fetchone()
        cursor.execute(f'SELECT Patient_name FROM patient WHERE Patient_no = "{patient_no}";')
        name = cursor.fetchone()
        mail.send_message(
            'The University of Lahore Teaching Hospital: Patient Discharge',
            sender=params['gmail_username'],
            recipients=[email[0]],
            body=f'Thank you for choosing The University of Lahore Teaching Hospital\n'
                 f'Patient No: {patient_no}\nPatient Name: {name[0]}\nDepartment: {department}\nRoom: {room}'
                 f'Room Charges: {room_charges}\nTest Charges: {test_charges}\nOperation Charges: {operation_charges}\n'
                 f'Blood Charges: {blood_charges}\nDoctor Charges: {doctor_charges}\nTotal Charges: {total_charges}\n'
                 f'Payment Status: {payment_status}\nDischarge Date: {datetime.now()}'
        )
        return render_template("patient_discharge.html", params=params, departments=departments, rooms=rooms)

    return render_template("patient_discharge.html", params=params, departments=departments, rooms=rooms)


@app.route("/doctor-prescription", methods=["GET", "POST"])
def d_prescription():
    if request.method == 'POST':
        patient_no = request.form.get("patient-no")
        prescription = request.form.get("prescription")
        entry = DoctorPrescription(Patient_no=patient_no, Prescription=prescription)
        db.session.add(entry)
        db.session.commit()
        cursor.execute(f'UPDATE `patient` SET `Prescription` = "{prescription}" WHERE'
                       f' `patient`.`Patient_no` = "{patient_no}";')
        connection.commit()
        cursor.execute(f'SELECT Email FROM patient WHERE Patient_no = "{patient_no}";')
        email = cursor.fetchone()
        cursor.execute(f'SELECT Patient_name FROM patient WHERE Patient_no = "{patient_no}";')
        name = cursor.fetchone()
        mail.send_message(
            'The University of Lahore Teaching Hospital: Doctor Prescription',
            sender=params['gmail_username'],
            recipients=[email[0]],
            body=f'Thank you for choosing The University of Lahore Teaching Hospital\n'
                 f'Patient No: {patient_no}\nPatient Name: {name[0]}\nDoctor Prescription: {prescription}'
        )
        return render_template("doctor_prescription.html", params=params)

    return render_template("doctor_prescription.html", params=params)


app.run(debug=True, port=5500)
