import collections
import datetime
from datetime import datetime, timedelta

from flask import Flask, request, jsonify, render_template, g
import sqlite3

from openpyxl.reader.excel import load_workbook

app = Flask(__name__)
DATABASE = 'defects.db'



def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    return db


@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()


@app.route('/')
def index():
    return render_template('index.html')


@app.route('/get_application_names', methods=['GET'])
def get_application_names():
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute('SELECT DISTINCT application_name FROM defects')
    applications = cursor.fetchall()
    conn.close()
    application_list = [app[0] for app in applications if app[0]]  # Access the first element of the tuple
    # Insert 'All' option at the beginning of the list
    #application_list.insert(0, 'All')
    return jsonify(application_list)

@app.route('/get_defects', methods=['GET','POST'])
def get_defects():
    db = get_db()
    cursor = db.cursor()

    query = 'SELECT * FROM defects WHERE 1=1'
    params = []

    if 'application_name' in request.args and request.args['application_name'] and request.args['application_name'].lower() != 'all':
        query += ' AND application_name = ?'
        params.append(request.args['application_name'])

    if 'severity' in request.args and request.args['severity'] and request.args['severity'].lower() != 'all':
        query += ' AND severity = ?'
        params.append(request.args['severity'])

    if 'status' in request.args and request.args['status'] and request.args['status'].lower() != 'all':
        query += ' AND status = ?'
        params.append(request.args['status'])

    if 'from_date' in request.args and request.args['from_date']:
        query += ' AND open_date >= ?'
        params.append(request.args['from_date'])

    if 'to_date' in request.args and request.args['to_date']:
        query += ' AND open_date <= ?'
        params.append(request.args['to_date'])

    cursor.execute(query, params)
    defects = cursor.fetchall()
    return jsonify([{
        'id': row[0],
        'defect_id': row[1],
        'summary': row[2],
        'description': row[3],
        'status': row[4],
        'priority': row[5],
        'severity': row[6],
        'owner': row[7],
        'open_date': row[8],
        'close_date': row[9],
        'environment': row[10],
        'application_name': row[11],
        'release_version': row[12],
        'reported_by': row[13],
        'reported_date': row[14],
        'steps_to_reproduce': row[15],
        'actual_result': row[16],
        'expected_result': row[17],
        'attachments': row[18],
        'comments': row[19],
        'iteration': row[20],
        'resolution': row[21]
    } for row in defects])

@app.route('/search_defects', methods=['POST'])
def search_defects():
    data = request.get_json()
    application_name = data.get('application_name')
    severity = data.get('severity')
    status = data.get('status')
    from_date = data.get('from_date')
    to_date = data.get('to_date')

    query = "SELECT * FROM defects WHERE 1=1"
    params = []

    if application_name and application_name.lower() != 'all':
        query += " AND application_name = ?"
        params.append(application_name)

    if severity and severity.lower() != 'all':
        query += " AND severity = ?"
        params.append(severity)

    if status and status.lower() != 'all':
        query += " AND status = ?"
        params.append(status)

    if from_date:
        query += " AND open_date >= ?"
        params.append(from_date)

    if to_date:
        query += " AND open_date <= ?"
        params.append(to_date)

    conn = get_db()
    cursor = conn.cursor()
    # print(query)
    cursor.execute(query, params)
    results = cursor.fetchall()

    # Get column names
    column_names = [description[0] for description in cursor.description]

    # Convert rows to list of dictionaries
    defects = [dict(zip(column_names, row)) for row in results]

    conn.close()

    return jsonify(defects)


@app.route('/add_defect', methods=['POST'])
def add_defect():
    data = request.get_json()
    db = get_db()
    cursor = db.cursor()
    cursor.execute('''
        INSERT INTO defects (
            defect_id, summary, description, status, priority, severity, owner, open_date,
            close_date, environment, application_name, release_version, reported_by, reported_date,
            steps_to_reproduce, actual_result, expected_result, attachments, comments, iteration, resolution
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    ''', (
        data['defect_id'], data['summary'], data['description'], data['status'], data['priority'], data['severity'],
        data['owner'], data['open_date'], data['close_date'], data['environment'], data['application_name'],
        data['release_version'], data['reported_by'], data['reported_date'], data['steps_to_reproduce'],
        data['actual_result'], data['expected_result'], data['attachments'], data['comments'], data['iteration'],
        data['resolution']
    ))
    db.commit()
    return jsonify({'status': 'success'})

def defect_exists(defect_id):
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM defects WHERE defect_id=?', (defect_id,))
    result = cursor.fetchone()
    conn.close()
    return result is not None

@app.route('/save_defect', methods=['POST'])
def save_defect():
    try:
        data = request.get_json()
        # print(data)
        defect_id = data.get('defect_id')
        summary = data.get('summary')
        description = data.get('description')
        status = data.get('status')
        priority = data.get('priority')
        severity = data.get('severity')
        owner = data.get('owner')
        open_date = data.get('open_date')
        close_date = data.get('close_date')
        environment = data.get('environment')
        application_name = data.get('application_name')
        release_version = data.get('release_version')
        reported_by = data.get('reported_by')
        reported_date = data.get('reported_date')
        steps_to_reproduce = data.get('steps_to_reproduce')
        actual_result = data.get('actual_result')
        expected_result = data.get('expected_result')
        attachments = data.get('attachments')
        comments = data.get('comments')
        iteration = data.get('iteration')
        resolution = data.get('resolution')

        conn = sqlite3.connect(DATABASE)
        cursor = conn.cursor()

        if defect_exists(defect_id):
            # Update existing defect
            cursor.execute('''
                UPDATE defects
                SET summary=?, description=?, status=?, priority=?, severity=?, owner=?, open_date=?, close_date=?, environment=?, application_name=?, release_version=?, reported_by=?, reported_date=?, steps_to_reproduce=?, actual_result=?, expected_result=?, attachments=?, comments=?, iteration=?, resolution=?
                WHERE defect_id=?
            ''', (summary, description, status, priority, severity, owner, open_date, close_date, environment, application_name, release_version, reported_by, reported_date, steps_to_reproduce, actual_result, expected_result, attachments, comments, iteration, resolution, defect_id))
        else:
            # Insert new defect
            cursor.execute('''
                INSERT INTO defects (defect_id, summary, description, status, priority, severity, owner, open_date, close_date, environment, application_name, release_version, reported_by, reported_date, steps_to_reproduce, actual_result, expected_result, attachments, comments, iteration, resolution)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (defect_id, summary, description, status, priority, severity, owner, open_date, close_date, environment, application_name, release_version, reported_by, reported_date, steps_to_reproduce, actual_result, expected_result, attachments, comments, iteration, resolution))

        conn.commit()
        conn.close()
        return '', 204
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/bulk_upload')
def bulk_upload_page():
    return render_template('bulk_upload.html')

@app.route('/bulk_upload', methods=['POST'])
def bulk_upload():
    if 'file' not in request.files:
        return 'No file part', 400
    file = request.files['file']
    if file.filename == '':
        return 'No selected file', 400

    try:
        # Load the uploaded Excel file
        workbook = load_workbook(file)
        sheet = workbook.active

        # Define the mapping between Excel columns and database columns
        column_mapping = {
            'Defect ID': 'defect_id',
            'Summary': 'summary',
            'Description': 'description',
            'Status': 'status',
            'Priority': 'priority',
            'Severity': 'severity',
            'Owner': 'owner',
            'Open Date': 'open_date',
            'Close Date': 'close_date',
            'Environment': 'environment',
            'Application Name': 'application_name',
            'Release Version': 'release_version',
            'Reported By': 'reported_by',
            'Reported Date': 'reported_date',
            'Steps to Reproduce': 'steps_to_reproduce',
            'Actual Result': 'actual_result',
            'Expected Result': 'expected_result',
            'Attachments': 'attachments',
            'Comments': 'comments',
            'Iteration': 'iteration',
            'Resolution': 'resolution'
        }

        # Get the header row
        header = [cell.value for cell in sheet[1]]

        # Check if all required columns are present
        for col in column_mapping.keys():
            if col not in header:
                return f'Missing required column: {col}', 400

        # Prepare data for insertion
        rows_to_insert = []
        conn = get_db()
        cursor = conn.cursor()

        for row in sheet.iter_rows(min_row=2, values_only=True):
            row_dict = {header[i]: row[i] for i in range(len(row))}
            row_data = tuple(row_dict.get(col) for col in column_mapping.keys())

            # Check if the defect ID already exists
            defect_id = row_dict.get('Defect ID')
            cursor.execute('SELECT id FROM defects WHERE defect_id = ?', (defect_id,))
            existing_record = cursor.fetchone()

            if existing_record:
                # Update existing record
                cursor.execute(f'''
                    UPDATE defects
                    SET summary=?, description=?, status=?, priority=?, severity=?, owner=?, open_date=?, close_date=?, environment=?, application_name=?, release_version=?, reported_by=?, reported_date=?, steps_to_reproduce=?, actual_result=?, expected_result=?, attachments=?, comments=?, iteration=?, resolution=?
                    WHERE defect_id=?
                ''', row_data[1:] + (defect_id,))
            else:
                # Insert new record
                cursor.execute(f'''
                    INSERT INTO defects ({', '.join(column_mapping.values())})
                    VALUES ({', '.join(['?'] * len(column_mapping))})
                ''', row_data)

        conn.commit()
        conn.close()

        return '', 204
    except Exception as e:
        return str(e), 500

@app.route('/delete_defect/<int:defect_id>', methods=['DELETE'])
def delete_defect(defect_id):
    db = get_db()
    cursor = db.cursor()
    cursor.execute('DELETE FROM defects WHERE id = ?', (defect_id,))
    db.commit()
    return jsonify({'status': 'success'})

@app.route('/analytics')
def analytics():
    conn = get_db()
    query = "SELECT DISTINCT application_name FROM defects"
    applications = conn.execute(query).fetchall()
    applications = [row[0] for row in applications]
    conn.close()

    filters = {
        'severity': request.form.get('severity'),
        'status': request.form.get('status'),
        'from_date': request.form.get('from_date'),
        'to_date': request.form.get('to_date'),
        'application_name': request.form.get('application_name')
    }
    return render_template('analytics.html', applications=applications, filters=filters)


# Function to convert rows to dictionaries
def rows_to_dicts(cursor, rows):
    columns = [col[0] for col in cursor.description]
    return [dict(zip(columns, row)) for row in rows]

# API to get filtered data
@app.route('/fetch_analytics_data', methods=['GET'])
def fetch_analytics_data():
    conn = get_db()
    query = "SELECT * FROM defects WHERE 1=1"
    filters = request.args

    if filters.get('severity'):
        query += f" AND severity='{filters['severity']}'"
    if filters.get('status'):
        query += f" AND status='{filters['status']}'"
    if filters.get('from_date'):
        query += f" AND open_date >= '{filters['from_date']}'"
    if filters.get('to_date'):
        query += f" AND open_date <= '{filters['to_date']}'"
    if filters.get('application_name'):
        query += f" AND application_name='{filters['application_name']}'"

    cursor = conn.cursor()
    cursor.execute(query)
    defects = cursor.fetchall()
    data = rows_to_dicts(cursor, defects)

    return jsonify(data)

if __name__ == '__main__':
    app.run(debug=True)

