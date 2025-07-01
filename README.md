NOTE FOR WSMB

Cipta VPC:

            Pergi ke perkhidmatan VPC. Klik "Create VPC".
            Pilih "VPC and more".
            Konfigurasi Wizard:
            Name tag auto-generation: nama-vpc
            Number of Availability Zones (AZs): Pilih 2.
            Number of public subnets: Pilih 2.
            Number of private subnets: Pilih 2.
            NAT gateways: Pilih "None".
            VPC endpoints: Pilih "S3 Gateway".
            Klik "Create VPC".
            
Cipta VPC Endpoint untuk Secrets Manager:

            Di panel navigasi kiri, klik "Endpoints" -> "Create Endpoint".
            Name tag: nama-endpoint
            Service category: "AWS services".
            Services: Cari dan pilih com.amazonaws.<region>.secretsmanager.
            VPC: Pilih nama-vpc anda.
            Subnets: Pilih kedua-dua subnet persendirian anda.
            Security groups: Pilih security group lalai (default) VPC anda.
            Klik "Create endpoint".

Pergi ke EC2, klik "Launch instances".

Name: nama-web-server | OS: Ubuntu | Instance type: t2.micro.

Key pair (login): Cipta atau pilih yang sedia ada.

Network settings -> Edit:

            VPC: nama-vpc | Subnet: Pilih salah satu subnet awam.
            Auto-assign public IP: Enable.
            Firewall (security groups): "Create security group" -> web-sg. Benarkan SSH (dari "My IP") dan HTTP (dari "Anywhere").
Kembangkan "Advanced details", tatal ke "User data".

Tampal skrip User Data lengkap di bawah:

            #!/bin/bash
            apt-get update -y
            apt-get install -y apache2 php libapache2-mod-php mysql-client
            systemctl start apache2
            systemctl enable apache2
            cat <<EOF > /var/www/html/index.html
            <!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>nama Live Dashboard</title><style>body{font-family:Arial,sans-serif;margin:20px;background-color:#f4f4f4}h1{color:#333}table{width:100%;border-collapse:collapse;margin-top:20px;box-shadow:0 2px 3px rgba(0,0,0,.1)}th,td{padding:12px;border:1px solid #ddd;text-align:left}th{background-color:#007bff;color:#fff}tr:nth-child(even){background-color:#f9f9f9}button{padding:10px 15px;font-size:16px;cursor:pointer;background-color:#28a745;color:#fff;border:none;border-radius:5px}#loading{display:none;color:#007bff}</style></head><body><h1>nama Live Telemetry Dashboard</h1><button onclick="fetchLogs()">Refresh Logs</button><p id="loading">Loading data...</p><table id="logsTable"><thead><tr><th>Fleet ID</th><th>Timestamp</th><th>Speed (km/h)</th><th>Location</th></tr></thead><tbody id="logsBody"></tbody></table><script>function fetchLogs(){const t=document.getElementById("logsBody"),e=document.getElementById("loading");t.innerHTML="",e.style.display="block",fetch("getlogs.php").then(t=>t.json()).then(o=>{let n;if(e.style.display="none",o&&"string"==typeof o.body)try{n=JSON.parse(o.body)}catch(a){return console.error("Failed to parse body JSON:",a),void(t.innerHTML='<tr><td colspan="4">Error: Invalid data format received.</td></tr>')}else n=o;if(n.error)return void(t.innerHTML='<tr><td colspan="4">Error: '+n.error+"</td></tr>");0===n.length?t.innerHTML='<tr><td colspan="4">No logs found.</td></tr>':n.forEach(e=>{let o=t.insertRow();o.insertCell(0).innerText=e.fleet_id,o.insertCell(1).innerText=new Date(e.log_timestamp).toLocaleString(),o.insertCell(2).innerText=e.speed,o.insertCell(3).innerText=e.location})}).catch(o=>{e.style.display="none",console.error("Error fetching logs:",o),t.innerHTML='<tr><td colspan="4">Failed to load data. Check console for details.</td></tr>'})}window.onload=fetchLogs;</script></body></html>
            EOF
            cat <<EOF > /var/www/html/getlogs.php
            <?php
            header("Content-Type: application/json");
            \$apiUrl = "PASTE_YOUR_API_GATEWAY_GET_URL_HERE";
            \$json_data = @file_get_contents(\$apiUrl);
            echo (\$json_data === FALSE) ? json_encode(["error" => "Failed to fetch from API Gateway. Check the URL in getlogs.php."]) : \$json_data;
            ?>
            EOF
            chown www-data:www-data /var/www/html/*
Klik "Launch instance".

Pergi ke RDS, klik "Create database".

Method: "Standard Create" | Engine: MySQL | Template: "Dev/Test".

Settings: nama-db, admin, tetapkan dan simpan kata laluan anda.

Availability and durability: Pilih "Create a standby instance" (Multi-AZ).

Connectivity:

            VPC: nama-vpc | Public access: "No".
            VPC security group: "Create new" -> rds-sg.
Additional configuration: Initial database name: nama.

Klik "Create database".

Konfigurasi Security Group rds-sg:

            Pergi ke Security Groups, pilih rds-sg.
            
                        Edit inbound rules. Tambah tiga peraturan berikut:
                        Type: HTTPS (port 443) | Source: rds-sg (dirinya sendiri). Ini untuk Lambda bercakap dengan VPC Endpoint.
                        Type: MySQL/Aurora (port 3306) | Source: rds-sg (dirinya sendiri). Ini untuk Lambda bercakap dengan RDS.
                        Type: MySQL/Aurora (port 3306) | Source: web-sg. Ini untuk EC2 bercakap dengan RDS semasa persediaan.
            Klik "Save rules".
Lampirkan Security Group pada VPC Endpoint:

            Pergi ke Endpoints, pilih secrets-manager-endpoint.
            Klik tab "Security Groups" -> "Manage security groups".
            Tandakan kotak untuk rds-sg (selain daripada yang default). Klik "Save changes".
Sambung & Cipta Jadual:

            SSH ke dalam EC2 nama-web-server anda.
            Dapatkan Endpoint pangkalan data RDS anda.
            Di terminal EC2, jalankan: mysql -h NAMA_ENDPOINT_RDS_ANDA -u admin -p -D nama
            Masukkan kata laluan RDS anda.
Di prompt mysql>, tampal dan jalankan:

            CREATE TABLE fleet_logs (fleet_id VARCHAR(255) NOT NULL, log_timestamp TIMESTAMP NOT NULL, speed INT, location VARCHAR(255), PRIMARY KEY (fleet_id, log_timestamp));
Sahkan dengan SHOW TABLES; dan kemudian exit.

Secrets Manager: Pergi ke AWS Secrets Manager, "Store a new secret".

Pilih "Other type of secret".

Di bawah "Key/value pairs", cipta kunci berikut:

            username -> admin
            password -> KATA_LALUAN_RDS_ANDA
            host -> ENDPOINT_RDS_ANDA
            dbname -> nama
Namakan rahsia nama/db/credentials. Salin ARN rahsia.

Peranan IAM: Pergi ke IAM -> Roles -> Create role.

Trusted entity: Lambda.

Permissions: Lampirkan polisi terurus AWSLambdaVPCAccessExecutionRole.

Namakan peranan nama-lambda-role.

Selepas dicipta, buka peranan itu dan "Add permissions" -> "Create inline policy". Gunakan tab JSON dan tampal kod berikut (gantikan placeholder):

            {"Version": "2012-10-17","Statement": [{"Effect": "Allow","Action": "secretsmanager:GetSecretValue","Resource": "ARN_SECRETS_MANAGER_ANDA"}]}

Lambda Layer untuk PyMySQL:

            Cipta direktori: mkdir -p python/lib/python3.9/site-packages.
            Pasang library: pip install PyMySQL -t python/lib/python3.9/site-packages.
            Zip direktori: zip -r pymysql_layer.zip python.
            Di konsol Lambda, pergi ke "Layers" -> "Create layer". Muat naik pymysql_layer.zip, pilih runtime Python 3.9.

Cipta dua fungsi Lambda: nama-post-log dan nama-get-logs.

Untuk kedua-duanya:

            Runtime: Python 3.9 | Role: nama-lambda-role.
            VPC: Pilih nama-vpc, kedua-dua subnet persendirian, dan security group rds-sg.
            Lampirkan Lambda Layer PyMySQL yang anda cipta.

Konfigurasi Asas Selepas Cipta:

            Untuk setiap fungsi, pergi ke tab "Configuration" -> "General configuration" -> "Edit".
            Tingkatkan Timeout kepada 15 saat.
            Pergi ke tab "Code". Di bawah "Runtime settings", klik "Edit".
            Tukar Handler kepada lambda_function.handler. Klik "Save".

Kod untuk nama-post-log: (Gantikan ARN)

            import json, pymysql, boto3, os
            SECRET_ARN = 'ARN_SECRETS_MANAGER_ANDA'
            secrets_manager = boto3.client('secretsmanager')
            def get_db_creds():
    response = secrets_manager.get_secret_value(SecretId=SECRET_ARN)
    return json.loads(response['SecretString'])
            def handler(event, context):
    try:
        creds = get_db_creds()
        conn = pymysql.connect(host=creds['host'], user=creds['username'], passwd=creds['password'], db=creds['dbname'], connect_timeout=5)
        body = json.loads(event['body'])
        with conn.cursor() as cursor:
            sql = "INSERT INTO fleet_logs (fleet_id, log_timestamp, speed, location) VALUES (%s, NOW(), %s, %s)"
            cursor.execute(sql, (body['fleet_id'], body['speed'], body['location']))
        conn.commit()
        conn.close()
        return {'statusCode': 201, 'headers': {'Access-Control-Allow-Origin': '*'}, 'body': json.dumps({'message': 'Log created successfully'})}
    except Exception as e:
        print(e)
        return {'statusCode': 500, 'headers': {'Access-Control-Allow-Origin': '*'}, 'body': json.dumps({'error': str(e)})}

Kod untuk nama-get-logs: (Gantikan ARN)

            import json, pymysql, boto3, os
            from datetime import datetime
            SECRET_ARN = 'ARN_SECRETS_MANAGER_ANDA'
            secrets_manager = boto3.client('secretsmanager')
            def get_db_creds():
    response = secrets_manager.get_secret_value(SecretId=SECRET_ARN)
    return json.loads(response['SecretString'])
            def datetime_converter(o):
    if isinstance(o, datetime): return o.isoformat()
            def handler(event, context):
    try:
        creds = get_db_creds()
        conn = pymysql.connect(host=creds['host'], user=creds['username'], passwd=creds['password'], db=creds['dbname'], connect_timeout=5, cursorclass=pymysql.cursors.DictCursor)
        with conn.cursor() as cursor:
            cursor.execute("SELECT fleet_id, log_timestamp, speed, location FROM fleet_logs ORDER BY log_timestamp DESC LIMIT 20")
            logs = cursor.fetchall()
        conn.close()
        return {'statusCode': 200, 'headers': {'Access-Control-Allow-Origin': '*', 'Content-Type': 'application/json'}, 'body': json.dumps(logs, default=datetime_converter)}
    except Exception as e:
        print(e)
        return {'statusCode': 500, 'headers': {'Access-Control-Allow-Origin': '*'}, 'body': json.dumps({'error': 'Failed to retrieve data from the database.'})}

Klik "Deploy" untuk kedua-dua fungsi selepas menampal kod.

Bina REST API namaAPI.

Cipta resource /logs.

Pada /logs, cipta method GET dan POST.

Konfigurasi Integrasi:

            Untuk method GET: Integrasikan dengan nama-get-logs. JANGAN tandakan kotak "Use Lambda Proxy integration".
            Untuk method POST: Integrasikan dengan nama-post-log. WAJIB tandakan kotak "Use Lambda Proxy integration".

Pilih resource /logs, Actions -> Enable CORS. Sahkan tetapan.

WAJIB: Actions -> Deploy API. Pilih stage v1. Salin Invoke URL untuk method GET dan POST.

SSH ke dalam EC2 nama-web-server.

Jalankan: sudo nano /var/www/html/getlogs.php.

Gantikan PASTE_YOUR_API_GATEWAY_GET_URL_HERE dengan URL Invoke GET yang sebenar.

Simpan fail.

Gunakan curl untuk menghantar data ke endpoint POST anda:

            curl -X POST 'URL_API_GATEWAY_POST_ANDA' -H 'Content-Type: application/json' -d '{"fleet_id": "WSD2025", "speed": 110, "location": "Cyberjaya"}'

Ambil Screenshots: Dashboard web yang berfungsi, konfigurasi Secrets Manager, CloudWatch Logs, rajah seni bina.

Salin URL & Kod: URL dashboard, URL API (POST & GET), kod sumber Lambda.

Tulis Penjelasan: Sediakan satu perenggan ringkas yang menerangkan kelebihan reka bentuk anda.
