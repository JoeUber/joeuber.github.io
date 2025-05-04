<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Church Volunteer Scheduler</title>
  <script src="https://cdn.jsdelivr.net/npm/react@18.2.0/umd/react.development.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/react-dom@18.2.0/umd/react-dom.development.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@babel/standalone@7.20.15/Babel.min.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
</head>
<body>
  <div id="root" class="container mx-auto p-4"></div>

  <script type="text/babel">
    const roles = ['Proclaimer (First Reader)', 'Proclaimer (Second Reader)', 'Sacristan', 'Eucharistic Minister'];
    const masses = ['5pm Saturday Mass', '9am Sunday Mass', '11am Sunday Mass'];

    function App() {
      const [schedule, setSchedule] = React.useState([]);
      const [volunteers, setVolunteers] = React.useState([]);
      const [file, setFile] = React.useState(null);
      const [absence, setAbsence] = React.useState({ volunteer: '', role: '', mass: '' });

      const handleFileUpload = (event) => {
        const file = event.target.files[0];
        setFile(file);
        const reader = new FileReader();
        reader.onload = (e) => {
          const data = new Uint8Array(e.target.result);
          const workbook = XLSX.read(data, { type: 'array' });
          const sheetName = workbook.SheetNames[0];
          const worksheet = workbook.Sheets[sheetName];
          const json = XLSX.utils.sheet_to_json(worksheet);
          const parsedSchedule = json.map(row => ({
            name: row.Name,
            email: row.Email,
            phone: row.Phone,
            role: row.Role,
            mass: row.Mass,
            date: row.Date
          }));
          setSchedule(parsedSchedule);
          const uniqueVolunteers = [...new Set(parsedSchedule.map(item => item.name))].map(name => ({
            name,
            email: parsedSchedule.find(item => item.name === name).email,
            phone: parsedSchedule.find(item => item.name === name).phone
          }));
          setVolunteers(uniqueVolunteers);
          // Export to Google Sheets
          exportToGoogleSheets(json);
        };
        reader.readAsArrayBuffer(file);
      };

      const exportToGoogleSheets = (data) => {
        const scriptURL = 'https://script.google.com/macros/s/AKfycbxMNittyhxgQT2wht3IQWzBjg8_zuJjjpVoVzoLBqKm75Hw6LuHGyZp_EXBcNdnCihijw/exec';
        fetch(scriptURL, {
          method: 'POST',
          body: JSON.stringify(data),
          headers: { 'Content-Type': 'application/json' }
        })
        .then(response => response.json())
        .then(data => console.log('Data sent to Google Sheets:', data))
        .catch(error => console.error('Error sending to Google Sheets:', error));
      };

      const sendNotifications = (mass, date) => {
        const scriptURL = 'YOUR_GOOGLE_APPS_SCRIPT_URL';
        const volunteersForMass = schedule.filter(item => item.mass === mass && item.date === date);
        fetch(scriptURL, {
          method: 'POST',
          body: JSON.stringify({ action: 'sendNotifications', volunteers: volunteersForMass, mass, date }),
          headers: { 'Content-Type': 'application/json' }
        })
        .then(response => response.json())
        .then(data => alert('Notifications sent successfully'))
        .catch(error => alert('Error sending notifications: ' + error.message));
      };

      const handleAbsence = (e) => {
        e.preventDefault();
        const { volunteer, role, mass } = absence;
        const date = new Date().toISOString().split('T')[0];
        const group = schedule.filter(item => item.role === role);
        const scriptURL = 'YOUR_GOOGLE_APPS_SCRIPT_URL';
        fetch(scriptURL, {
          method: 'POST',
          body: JSON.stringify({ action: 'sendAbsence', volunteer, role, mass, date, group }),
          headers: { 'Content-Type': 'application/json' }
        })
        .then(response => response.json())
        .then(data => {
          alert(`Absence notification sent for ${volunteer}`);
          setAbsence({ volunteer: '', role: '', mass: '' });
        })
        .catch(error => alert('Error sending absence notification: ' + error.message));
      };

      return (
        <div className="space-y-6">
          <h1 className="text-3xl font-bold text-center">Church Volunteer Scheduler</h1>
          
          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold mb-4">Upload Schedule (Excel)</h2>
            <input
              type="file"
              accept=".xlsx, .xls"
              onChange={handleFileUpload}
              className="block w-full text-sm text-gray-500
                file:mr-4 file:py-2 file:px-4
                file:rounded-full file:border-0
                file:text-sm file:font-semibold
                file:bg-blue-50 file:text-blue-700
                hover:file:bg-blue-100"
            />
          </div>

          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold mb-4">Current Schedule</h2>
            <div className="grid grid-cols-1 gap-4">
              {masses.map(mass => (
                <div key={mass} className="border p-4 rounded">
                  <h3 className="text-lg font-medium">{mass}</h3>
                  {roles.map(role => (
                    <div key={role}>
                      <p className="font-semibold">{role}:</p>
                      {schedule
                        .filter(item => item.mass === mass && item.role === role)
                        .map((item, idx) => (
                          <p key={idx}>{item.name} ({item.date})</p>
                        ))}
                    </div>
                  ))}
                  <button
                    onClick={() => sendNotifications(mass, new Date().toISOString().split('T')[0])}
                    className="mt-2 bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600"
                  >
                    Send Weekly Notifications
                  </button>
                </div>
              ))}
            </div>
          </div>

          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold mb-4">Report Absence</h2>
            <form onSubmit={handleAbsence} className="space-y-4">
              <div>
                <label className="block text-sm font-medium text-gray-700">Volunteer</label>
                <select
                  value={absence.volunteer}
                  onChange={(e) => setAbsence({ ...absence, volunteer: e.target.value })}
                  className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
                >
                  <option value="">Select Volunteer</option>
                  {volunteers.map(vol => (
                    <option key={vol.name} value={vol.name}>{vol.name}</option>
                  ))}
                </select>
              </div>
              <div>
                <label className="block text-sm font-medium text-gray-700">Role</label>
                <select
                  value={absence.role}
                  onChange={(e) => setAbsence({ ...absence, role: e.target.value })}
                  className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
                >
                  <option value="">Select Role</option>
                  {roles.map(role => (
                    <option key={role} value={role}>{role}</option>
                  ))}
                </select>
              </div>
              <div>
                <label className="block text-sm font-medium text-gray-700">Mass</label>
                <select
                  value={absence.mass}
                  onChange={(e) => setAbsence({ ...absence, mass: e.target.value })}
                  className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
                >
                  <option value="">Select Mass</option>
                  {masses.map(mass => (
                    <option key={mass} value={mass}>{mass}</option>
                  ))}
                </select>
              </div>
              <button
                type="submit"
                className="bg-red-500 text-white py-2 px-4 rounded hover:bg-red-600"
              >
                Report Absence
              </button>
            </form>
          </div>
        </div>
      );
    }

    ReactDOM.render(<App />, document.getElementById('root'));
  </script>
</body>
</html>
