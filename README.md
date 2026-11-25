# Vitalyx-
VITALYX Care is a simple personal tool for managing your daily medicine routine. Add your medicines, see today's schedule at a glance, mark doses as taken, and export or back up your history whenever you like. Nothing is sent to a server, and everything is stored locally in your browser. 
let medicines = JSON.parse(
    localStorage.getItem("vitalyxMedicines") || "[]"
);

let history = JSON.parse(
    localStorage.getItem("vitalyxHistory") || "[]"
);

function saveData() {
    localStorage.setItem(
        "vitalyxMedicines",
        JSON.stringify(medicines)
    );

    localStorage.setItem(
        "vitalyxHistory",
        JSON.stringify(history)
    );

    render();
}

function render() {
    const list = document.getElementById("medicineList");

    list.innerHTML = "";

    medicines.forEach((medicine, index) => {
        const taken = history.some(
            item => item.id === medicine.id &&
            item.date === new Date().toLocaleDateString()
        );

        list.innerHTML += `
            <div class="medicine">
                <h3>${escapeHTML(medicine.name)}</h3>
                <p>Dose: ${escapeHTML(medicine.dose)}</p>
                <p>Time: ${escapeHTML(medicine.time)}</p>
                <p>Stock: ${medicine.stock}</p>

                <button onclick="markTaken('${medicine.id}')"
                    ${taken ? "disabled" : ""}>
                    ${taken ? "Taken ✓" : "Mark Taken"}
                </button>

                <button class="secondary"
                    onclick="deleteMedicine('${medicine.id}')">
                    Delete
                </button>
            </div>
        `;
    });

    const today = new Date().toLocaleDateString();

    const takenToday = medicines.filter(m =>
        history.some(h => h.id === m.id && h.date === today)
    ).length;

    document.getElementById("totalMedicines").textContent =
        medicines.length;

    document.getElementById("takenToday").textContent =
        takenToday;

    document.getElementById("remaining").textContent =
        medicines.length - takenToday;

    document.getElementById("history").innerHTML =
        history.slice(-10).reverse().map(item =>
            `<p>${escapeHTML(item.name)} — ${escapeHTML(item.date)} ${escapeHTML(item.time)}</p>`
        ).join("") || "No medicines taken yet.";
}

function escapeHTML(value) {
    return String(value).replace(/[&<>"']/g, c => ({
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#39;"
    })[c]);
}

document.getElementById("medicineForm").addEventListener(
    "submit", function(event) {
        event.preventDefault();

        const medicine = {
            id: crypto.randomUUID(),
            name: document.getElementById("medicineName").value.trim(),
            dose: document.getElementById("medicineDose").value.trim(),
            time: document.getElementById("medicineTime").value,
            stock: Number(document.getElementById("medicineStock").value)
        };

        medicines.push(medicine);

        this.reset();

        saveData();
    }
);

function markTaken(id) {
    const medicine = medicines.find(m => m.id === id);

    if (!medicine) return;

    const date = new Date().toLocaleDateString();

    if (history.some(h => h.id === id && h.date === date)) return;

    history.push({
        id,
        name: medicine.name,
        date,
        time: new Date().toLocaleTimeString()
    });

    if (medicine.stock > 0) {
        medicine.stock--;
    }

    saveData();
}

function deleteMedicine(id) {
    if (!confirm("Delete this medicine?")) return;

    medicines = medicines.filter(m => m.id !== id);

    history = history.filter(h => h.id !== id);

    saveData();
}

async function enableNotifications() {
    if (!("Notification" in window)) {
        alert("Notifications are not supported.");
        return;
    }

    const permission = await Notification.requestPermission();

    alert(
        permission === "granted"
        ? "Notifications enabled!"
        : "Notification permission not granted."
    );
}

function checkReminders() {
    if (!("Notification" in window)) return;
    if (Notification.permission !== "granted") return;

    const currentTime = new Date().toTimeString().slice(0, 5);
    const today = new Date().toLocaleDateString();

    medicines.forEach(medicine => {
        const alreadyTaken = history.some(
            h => h.id === medicine.id && h.date === today
        );

        const key = `vitalyx-${medicine.id}-${today}-${currentTime}`;

        if (
            medicine.time === currentTime &&
            !alreadyTaken &&
            !sessionStorage.getItem(key)
        ) {
            new Notification("VITALYX Medicine Reminder", {
                body: `Time for ${medicine.name}`
            });

            sessionStorage.setItem(key, "sent");
        }
    });
}

setInterval(checkReminders, 30000);

function exportData() {
    const rows = [
        ["Medicine", "Dose", "Time", "Stock"],
        ...medicines.map(m => [
            m.name, m.dose, m.time, m.stock
        ])
    ];

    const csv = rows.map(row =>
        row.map(value =>
            `"${String(value).replace(/"/g, '""')}"`
        ).join(",")
    ).join("\n");

    const file = new Blob([csv], {type: "text/csv"});

    const link = document.createElement("a");

    link.href = URL.createObjectURL(file);
    link.download = "vitalyx-medicines.csv";
    link.click();

    URL.revokeObjectURL(link.href);
}

render();
