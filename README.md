# Advanced-Disk-Scheduling-Simulator
// Disk scheduling simulator using Flask with FCFS, SSTF, SCAN, C-SCAN.
from flask import Flask, request, jsonify, render_template
import matplotlib.pyplot as plt
import os

app = Flask(__name__)

HTML_PAGE = """ 
<!DOCTYPE html>
<html>
<head>
    <title>Disk Scheduling Simulator</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            text-align: center;
            padding: 20px;
        }
        .container {
            background: white;
            padding: 20px;
            max-width: 500px;
            margin: auto;
            box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.2);
            border-radius: 10px;
        }
        h1 {
            color: #333;
        }
        label {
            font-weight: bold;
        }
        input, select {
            width: 100%;
            padding: 8px;
            margin: 10px 0;
            border-radius: 5px;
            border: 1px solid #ccc;
        }
        button {
            background: #007BFF;
            color: white;
            padding: 10px;
            border: none;
            width: 100%;
            cursor: pointer;
            border-radius: 5px;
            font-size: 16px;
        }
        button:hover {
            background: #0056b3;
        }
        .result {
            margin-top: 20px;
            padding: 10px;
            background: #e9f5ff;
            border-radius: 5px;
        }
        img {
            margin-top: 20px;
            width: 100%;
            max-width: 500px;
            border-radius: 5px;
        }
    </style>
    <script>
        function runScheduler() {
            const requests = document.getElementById("requests").value;
            const head = document.getElementById("head").value;
            const algo = document.getElementById("algorithm").value;

            fetch("/schedule", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ requests: requests, head: head, algorithm: algo })
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById("result").innerHTML = 
                    "<strong>Seek Sequence:</strong> " + data.seek_sequence.join(" → ") + 
                    "<br><strong>Total Seek Time:</strong> " + data.seek_time +
                    "<br><strong>Average Seek Time:</strong> " + data.avg_seek_time.toFixed(2) +
                    "<br><strong>System Throughput:</strong> " + data.throughput.toFixed(2) + " requests/unit time";

                document.getElementById("result").style.display = "block";

                document.getElementById("seekGraph").src = "/static/seek_sequence.png";
                document.getElementById("seekGraph").style.display = "block";
            });
        }
    </script>
</head>
<body>
    <div class="container">
        <h1>Disk Scheduling Simulator</h1>
        <label>Enter Disk Requests (comma-separated):</label>
        <input type="text" id="requests" value="98,183,37,122">

        <label>Enter Initial Head Position:</label>
        <input type="number" id="head" value="53">

        <label>Select Algorithm:</label>
        <select id="algorithm">
            <option value="FCFS">FCFS</option>
            <option value="SSTF">SSTF</option>
            <option value="SCAN">SCAN</option>
            <option value="C-SCAN">C-SCAN</option>
        </select>

        <button onclick="runScheduler()">Run Scheduler</button>

        <div id="result" class="result" style="display: none;"></div>

        <img id="seekGraph" style="display: none;" />
    </div>
</body>
</html>
"""

def calculateMetrics(seekSequence, seekTime):
    numRequests = len(seekSequence) - 1
    avg_seek_time = seekTime / numRequests if numRequests > 0 else 0
    throughput = numRequests / seekTime if seekTime > 0 else 0
    return avg_seek_time, throughput

def generateGraph(seekSequence):
    plt.figure(figsize=(8, 5))
    plt.plot(seekSequence, range(len(seekSequence)), marker='o', linestyle='-', color='b', markersize=8)
    plt.xlabel("Cylinder Number")
    plt.ylabel("Seek Steps")
    plt.title("Disk Seek Sequence Visualization")
    plt.grid(True)

    # Save the plot
    if not os.path.exists("static"):
        os.makedirs("static")
    plt.savefig("static/seek_sequence.png")
    plt.close()

def fcfs(requestss, head):
    seekSequence = [head] + requestss
    seek_time = sum(abs(seekSequence[i] - seekSequence[i+1]) for i in range(len(seekSequence) - 1))
    return seekSequence, seek_time

def sstf(requestss, headd):
    seek_sequence = []
    total_seek_time = 0
    requests = sorted(requestss, key=lambda x: abs(x - headd))

    while requests:
        nextRequest = requests.pop(0)
        seek_sequence.append(nextRequest)
        total_seek_time += abs(headd - nextRequest)
        headd = nextRequest

    return [headd] + seek_sequence, total_seek_time

def scan(requestss, headd):
    requestss.sort()
    left = [r for r in requestss if r < headd]
    right = [r for r in requestss if r >= headd]

    seekSequence = [headd] + right + left[::-1]
    seekTime = sum(abs(seekSequence[i] - seekSequence[i+1]) for i in range(len(seekSequence) - 1))

    return seekSequence, seekTime

def cScan(requestss, headd):
    requestss.sort()
    right = [r for r in requestss if r >= headd]
    left = [r for r in requestss if r < headd]

    seekSequence = [headd] + right + left
    seek_time = sum(abs(seekSequence[i] - seekSequence[i+1]) for i in range(len(seekSequence) - 1))

    return seekSequence, seek_time

@app.route('/')
def index():
    return HTML_PAGE

@app.route('/schedule', methods=['POST'])
def schedule():
    data = request.json
    requests = list(map(int, data['requests'].split(',')))
    head = int(data['head'])
    algorithm = data['algorithm']

    if algorithm == "FCFS":
        seekSequence, seek_time = fcfs(requests, head)
    elif algorithm == "SSTF":
        seekSequence, seek_time = sstf(requests, head)
    elif algorithm == "SCAN":
        seekSequence, seek_time = scan(requests, head)
    elif algorithm == "C-SCAN":
        seekSequence, seek_time = cScan(requests, head)
    else:
        return jsonify({"error": "Invalid algorithm selected!"}), 400

    avg_seek_time, throughput = calculateMetrics(seekSequence, seek_time)

    # Generate seek sequence graph
    generateGraph(seekSequence)

    return jsonify({
        "seek_sequence": seekSequence,
        "seek_time": seek_time,
        "avg_seek_time": avg_seek_time,
        "throughput": throughput
    })

if __name__ == '__main__':
    app.run(debug=True)
