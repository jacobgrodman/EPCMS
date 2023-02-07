const activeHeading = document.querySelector("#activeHeading");
const active = document.querySelector("#active");

const templateRow = document.querySelector("#row");
const sort = document.querySelector("#sort");
const arrow = document.querySelector("#arrow");
const az = document.querySelector("#az");

//const specialty = document.querySelector("#specialty");
//const submitFilter = document.querySelector("#submitFilter");
//const clear = document.querySelector("#clear");

var filtersCreated = false;

var db = firebase.firestore().collection("members");
var whereQuery = {};

var currentSorting = "Z-A";
var specialtyList = [];

var adminAccess = false;
var invoiceButtons = [];

//BuildQuery(false);
HideAdminData();

function HideAdminData() {
    activeHeading.style.display = "none";
    active.style.display = "none";
}

firebase.auth().onAuthStateChanged((user) => {
    if (user) {
        firebase.firestore().collection("members").doc(user.uid).get().then((doc) => {
            if (doc.exists) {
                var userData = doc.data();
                if (userData["accessLevel"] == 0 || userData["accessLevel"] == 1) {
                    location.href = "/public-directory";
                }
                else if (userData["accessLevel"] == 3) {
                    adminAccess = true;
                }
            }
        }).then(() => {
            BuildDirectory();
        });
    }
    else {
        location.href = "/public-directory";
    }
});

function BuildDirectory() {
    BuildQuery(false);
    SwapSortButton();
    FillDirectory();
}

function BuildQuery(filterButtonPressed) {
    db = firebase.firestore().collection("members");
    if (currentSorting == "A-Z") {
        db = db.orderBy("lastName", "desc");
    }
    else {
        db = db.orderBy("lastName");
    }

    if (filterButtonPressed) {
        for (var key in whereQuery) {
            db = db.where(key, "==", whereQuery[key]);
        }
    }
}

function SwapSortButton() {

    if (currentSorting == "A-Z") {
        currentSorting = "Z-A";
        arrow.style.transform = 'rotate(-90deg)';
    }
    else {
        currentSorting = "A-Z";
        arrow.style.transform = 'rotate(90deg)';
    }

    az.innerHTML = currentSorting;
}

function RemoveRows() {
    var rows = document.getElementsByClassName('table-row');
    while (rows.length > 1) {
        rows[1].remove();
    }
}

function FillDirectory() {
    templateRow.style.display = "flex";
    RemoveRows();

    //CheckFilters();
    db.get().then(querySnapshot => {
        querySnapshot.forEach((doc) => {
            CreateRow(doc);

            // Check if the "specialty" field is not in the specialtyList array
            if (!specialtyList.includes(doc.data().primarySpecialty)) {
                // If not, add the value of the "specialty" field to the specialtyList array
                specialtyList.push(doc.data().primarySpecialty);
            }
        });
    }).then(() => {
        templateRow.style.display = "none";
        AddButtonsToArray();
        //if (!filtersCreated) {
        //    CreateFilters();
        //    filtersCreated = true;
        //}
    })
}

function CreateRow(doc) {
    var userData = doc.data();
    var newRow = templateRow.cloneNode(true);
    newRow.id = doc.id;

    var elements = RowElements(newRow);

    var checkboxID = "check-" + userData["email"];
    elements["checkbox"].setAttribute('id', checkboxID);

    var url = "/profile?uid=" + doc.id;
    newRow.childNodes[1].setAttribute('href', url);

    var fullName = userData['lastName'] + ", " + userData['firstName'];
    elements["fullName"].innerHTML = fullName;

    elements["specialty"].innerHTML = userData["primarySpecialty"]

    elements["email"].innerHTML = userData["email"];

    elements["company"].innerHTML = userData["company"];

    elements["website"].innerHTML = userData["companyURL"]
    elements["website"].setAttribute('href', userData["companyURL"]);

    var address = BuildAddress(userData);
    elements["address"].innerHTML = address;

    elements["phone"].innerHTML = userData["phone"];

    if (adminAccess == true) {
        activeHeading.style.display = "block";

        var expiryDate = new Date(userData["expires"]);
        var currentDate = new Date();

        elements["expires"].innerHTML = expiryDate.toLocaleDateString();
        active.style.display = "block";

        if (expiryDate < currentDate) {
            var buttonId = "invoice-" + doc.id;
            elements["invoiceButton"].setAttribute('id', buttonId);
        }
        else if (expiryDate >= currentDate){
            //children[9].childNodes[0].childNodes[1].style.display = "none";
            
            var buttonId = "invoice-" + uid;
            elements["invoiceButton"].setAttribute('id', buttonId);
        }
    }

    templateRow.after(newRow);
}

function RowElements(newRow) {
    var elements = {
        checkbox: newRow.childNodes[0].childNodes[0].childNodes[0],
        fullName: newRow.childNodes[1].childNodes[1],
        specialty: newRow.childNodes[2].childNodes[0],
        email: newRow.childNodes[3].childNodes[0],
        company: newRow.childNodes[4].childNodes[0],
        website: newRow.childNodes[5].childNodes[0],
        address: newRow.childNodes[6].childNodes[0],
        phone: newRow.childNodes[7].childNodes[0],
        expires: newRow.childNodes[8].childNodes[0].childNodes[0],
        invoiceButton: newRow.childNodes[8].childNodes[0].childNodes[1]
    };

    return elements;
}

function BuildAddress(userData) {
    var address = userData["primaryAddress"];

    if (userData["primaryAddress" != ""]) {
        if (userData["primaryAddress2"] != "") {
            address += ", " + userData["primaryAddress2"];
        }

        if (userData["primaryAddress3"] != "") {
            address += ", " + userData["primaryAddress3"];
        }

        address += "\n" + userData["primaryCity"] + ", " + userData["primaryState"] + " " + userData["primaryZip"]

        return address;
    }
    else {
        return "";
    }
}

function AddButtonsToArray() {
    let invoiceButtons = document.getElementsByClassName("invoice");
    for (let i = 0; i < invoiceButtons.length; i++) {
        invoiceButtons[i].addEventListener('click', function(event) {
            // TODO: get some html to format this email.
            SendEmail(event, false);
        });
    }
}

function SendEmail(event, mass) {
    var email = {};
    var message = {};
    var recipient;
    var userData;

    var memberID = event.target.id.replace("invoice-","");
    var member = firebase.firestore().collection("members").doc(memberID);
    
    if (mass == true) {
        recipient = MassEmailGet();
    }
    else {
        member.get().then((doc) => {
            if (doc.exists) {
                userData = doc.data();
                recipient = "jacob@jacobgrodman.com; avidwriter117@gmail.com;"; //userData["email"] + "; epcms@epcms.org;";
                message["html"] = ConstructHTML(userData);
                message["subject"] = "Your EPCMS Membership has Expired";
                message["text"] = "";
            }
        }).then(() => {
            email["to"] = recipient;
            email["message"] = message;
            firebase.firestore().collection("mail").doc().set(email);
        });
    }
}

function MassEmailGet() {
    var recipient;
    let memberCheckboxes = document.getElementsByClassName("memberCheckbox");
    for (let i = 0; i < memberCheckboxes.length; i++) {
        if (memberCheckboxes[i].checked) {
            var id = memberCheckboxes[i].id.replace("check-", "");
            recipient = recipient + " " + id + ";"
        }
    }
    console.log()
}

function ConstructHTML(userData) {
    var html = "Your epcms membership has expired.\n<a href='https://buy.stripe.com/cN214AfzmelI9S89AA'>Click Here to renew</a>";
    return html;
}

function CheckFilters() {
    let radio = document.querySelectorAll("input[type='radio']");
    var checkedRadio;

    for (var i = 0; i < radio.length; i++) {
        if (radio[i].checked) {
            checkedRadio = radio[i];
        }
    }

    if (checkedRadio != null) {
        whereQuery["primarySpecialty"] = checkedRadio.id;
        BuildQuery(true);
    }
    else {
        BuildQuery(false);
    }
}

function CreateFilters() {
    for (let i = 0; i < specialtyList.length; i++) {
        let wrapper = document.createElement("div");
        let radio = document.createElement("input");
        let label = document.createElement("label");

        wrapper.classList.add("checkboxWrapper");
        wrapper.setAttribute("style", "display: flex;")
        radio.type = "radio";
        radio.id = specialtyList[i];
        radio.name = "specialty";
        label.setAttribute("for", radio.id);
        label.innerHTML = specialtyList[i];

        wrapper.appendChild(radio);
        wrapper.appendChild(label);

        specialty.parentElement.nextElementSibling.appendChild(wrapper);
    }
}

function ToggleFilterCategory() {
    var checkboxes = specialty.parentElement.nextElementSibling;
    if (checkboxes.style.display === "block") {
        checkboxes.style.display = "none";
    }
    else {
        checkboxes.style.display = "block";
    }
}

// TODO: this sometimes doesn't refresh properly.
function ClearFilters() {
    let radio = document.querySelectorAll("input[type='radio']");
    radio.forEach((button) => {
        button.checked = false;
        FillDirectory();
    })
}

sort.addEventListener('click', BuildDirectory);
//specialty.addEventListener('click', ToggleFilterCategory);
//submitFilter.addEventListener('click', FillDirectory);
//clear.addEventListener('click', ClearFilters);