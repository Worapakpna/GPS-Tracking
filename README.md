<?php
    $apiKey = "AIzaSyAU-yyvFC4_i92oeRhwqz_iIiJb0ZotST8";
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PPAO SMART BUS Map</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons/font/bootstrap-icons.css" rel="stylesheet">
    <script src="https://maps.googleapis.com/maps/api/js?key=<?php echo $apiKey; ?>&libraries=places,drawing,geometry"></script>
    <style>
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: linear-gradient(to bottom, #00A9E0, #00457C); /* สีฟ้าไล่โทนไปสีกรมท่า */
            color: white;
        }

        .sidebar {
            width: 250px;
            background: linear-gradient(to bottom, #00457C, #001f3d); /* สีฟ้าเข้มไปสีกรมท่า */
            position: fixed;
            top: 0;
            left: 0;
            bottom: 0;
            padding: 10px;
            color: white;
            transition: transform 0.3s ease; /* ใช้สำหรับการแสดง/ซ่อนแถบ */
        }

        .sidebar.hidden {
            transform: translateX(-100%); /* ซ่อนแถบด้านข้าง */
        }

        .sidebar h2 {
            font-size: 22px;
            text-align: center;
            margin-bottom: 20px;
        }

        .menu-item {
            padding: 10px;
            cursor: pointer;
            display: flex;
            align-items: center;
            transition: background-color 0.3s ease;
        }

        .menu-item i {
            margin-right: 10px;
        }

        .menu-item:hover {
            background: #001f3d; /* สีเข้มเมื่อ hover */
            border-radius: 5px;
        }

        #map {
            margin-left: 250px;
            height: 100vh;
            width: calc(100% - 250px);
            transition: margin-left 0.3s ease; /* สำหรับการย้ายแผนที่เมื่อแถบถูกซ่อน */
        }

        .hidden-map {
            margin-left: 0;
            width: 100%; /* เมื่อแถบด้านข้างซ่อน แผนที่จะขยายเต็มหน้าจอ */
        }

        .button-container {
    position: fixed;
    top: 10px;
    left: 250px; /* ย้ายปุ่มให้ติดกับแถบข้าง */
    z-index: 1000;
}

.button-container button {
    padding: 12px 20px;
    font-size: 14px;
    cursor: pointer;
    margin-right: 10px;
    background-color: #006F97; /* สีฟ้าเข้ม */
    color: white;
    border: none;
    border-radius: 30px; /* ทำให้ปุ่มมีมุมมน */
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); /* เพิ่มเงาให้ปุ่มดูลอยขึ้น */
    transition: background-color 0.3s ease, transform 0.2s, box-shadow 0.3s ease;
}

.button-container button:hover {
    background-color: #00457C; /* สีกรมท่าเมื่อ hover */
    transform: scale(1.05); /* ขยายขนาดเมื่อ hover */
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.2); /* เพิ่มเงาเมื่อ hover */
}

.button-container button:active {
    background-color: #003a5a; /* สีกรมท่าลึกเมื่อกด */
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* เงาเบาๆ เมื่อกด */
}


        .search-dropdown {
            display: none;
            padding-left: 20px;
            justify-content: flex-start;
        }

        .search-dropdown {
            display: flex;
            width: 100%;
        }

        .search-dropdown input[type="text"] {
            padding: 10px;
            width: 180px;
            border-radius: 5px 0 0 5px;
            border: 1px solid #ccc;
            flex: 1;
        }

        .search-dropdown button {
            padding: 10px 15px;
            background-color: #00A9E0; /* สีฟ้าอ่อน */
            color: white;
            border: 1px solid #ccc;
            border-radius: 0 5px 5px 0;
            cursor: pointer;
        }

        .search-dropdown button:hover {
            background-color: #007ab8; /* สีฟ้าเข้มเมื่อ hover */
        }
    </style>
</head>
<body>
    <div class="button-container">
        <button onclick="toggleSidebar()">เปิด/ปิดแถบข้าง</button>
    </div>

    <div class="sidebar" id="sidebar">
        <h2><i class="bi bi-bus-front"></i> PPAO SMART BUS</h2>
        <div class="menu-item" onclick="toggleSearchDropdown()"><i class="bi bi-search"></i> ค้นหาสถานที่ / เส้นทาง</div>
        <div class="search-dropdown" id="searchDropdown">
            <input type="text" id="locationInput" placeholder="ค้นหาสถานที่...">
            <button onclick="searchLocation()">ค้นหา</button>
        </div>
        <div class="menu-item"><i class="bi bi-bus-front"></i> ข้อมูลรถ</div>
        <div class="menu-item" onclick="toggleRouteOptions()"><i class="bi bi-route"></i> ข้อมูลเส้นทาง</div>
        <div class="route-options" id="routeOptions" style="display: none;">
            <label><input type="checkbox" onclick="toggleRoute(1)" checked> สายที่ 1 </label><br/>
            <label><input type="checkbox" onclick="toggleRoute(2)"> สายที่ 2 </label><br/>
            <label><input type="checkbox" onclick="toggleRoute(3)"> สายที่ 3</label><br/>
        </div>
        <div class="menu-item"><i class="bi bi-person-walking"></i> ค้นหาเส้นทางเดินรถ</div>
    </div>

    <div id="map"></div>

    <script>
        var map, geocoder, currentLocationMarker, directionsService, directionsRenderer;
var busMarkers = []; // เก็บ markers ของป้ายรถ
var closestMarker = null; // เก็บ marker ที่ใกล้ที่สุด

function toggleSidebar() {
    var sidebar = document.querySelector(".sidebar");
    var mapContainer = document.getElementById("map");

    // Toggle sidebar visibility
    if (sidebar.style.display === "none") {
        sidebar.style.display = "block"; // แสดงแถบข้าง
        mapContainer.style.marginLeft = "250px"; // ขยายแผนที่กลับมา
        mapContainer.style.width = "calc(100% - 250px)"; // กำหนดขนาดแผนที่ให้ถูกต้อง
    } else {
        sidebar.style.display = "none"; // ซ่อนแถบข้าง
        mapContainer.style.marginLeft = "0"; // ทำให้แผนที่เต็มหน้าจอ
        mapContainer.style.width = "100%"; // ขยายแผนที่ให้เต็มจอ
    }
    // รีเซ็ต marker ที่ใกล้ที่สุดเมื่อแถบข้างเปิดหรือปิด
    highlightClosestBusStop();
}

function initMap() {
    map = new google.maps.Map(document.getElementById('map'), {
        center: { lat: 7.8804, lng: 98.3923 },
        zoom: 14
    });

    geocoder = new google.maps.Geocoder();
    directionsService = new google.maps.DirectionsService();
    directionsRenderer = new google.maps.DirectionsRenderer({ map: map });

    var busStops = [
        { lat: 7.872867, lng: 98.414099},
        { lat: 7.881441, lng: 98.40843 },
        { lat: 7.882448, lng: 98.404505 },
        { lat: 7.881315, lng: 98.40187 },
        { lat: 7.882689, lng: 98.399375 },
        { lat: 7.883056, lng: 98.395663 },
        { lat: 7.885406, lng: 98.39304 },
        { lat: 7.88602, lng: 98.392439 },
        { lat: 7.886501, lng: 98.390411 },
        { lat: 7.892385, lng: 98.38965 },
        { lat: 7.896248, lng: 98.388122 },
        { lat: 7.89607, lng: 98.386443 },
        { lat: 7.895252, lng: 98.385122 },
        { lat: 7.892285, lng: 98.385826 },
        { lat: 7.89066, lng: 98.385894 },
        { lat: 7.888741, lng: 98.384349 },
        { lat: 7.888096, lng: 98.380553 },
        { lat: 7.887603, lng: 98.377115 },
        { lat: 7.887616, lng: 98.371966 },
        { lat: 7.893048, lng: 98.368846 },
        { lat: 7.895991, lng: 98.368527 },
        { lat: 7.90503, lng: 98.364681 }
    ];

    var busIcon = {
        url: 'phuket_bus-stop_green-1.png',
        scaledSize: new google.maps.Size(35, 35) // ขนาดเริ่มต้น
    };

    // สร้าง marker สำหรับป้ายรถทุกตัว
    busStops.forEach(function(stop) {
        var marker = new google.maps.Marker({
            position: stop,
            map: map,
            icon: busIcon
        });
        busMarkers.push(marker); // เก็บ markers ทุกตัวไว้ใน array
    });

    if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(function(position) {
            var userLat = position.coords.latitude;
            var userLng = position.coords.longitude;
            var userLocation = { lat: userLat, lng: userLng };

            // ไอคอนของตำแหน่งผู้ใช้
            var userIcon = {
                path: 'M12 12c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4z',
                scale: 2,
                strokeColor: 'black',
                fillColor: '#FF0000',
                fillOpacity: 1,
            };

            currentLocationMarker = new google.maps.Marker({
                map: map,
                position: userLocation,
                icon: userIcon,
                label: {
                    text: "จุดที่คุณอยู่",
                    fontWeight: "bold",
                    fontSize: "14px",
                    color: "black",
                }
            });

            map.setCenter(userLocation); // กำหนดจุดศูนย์กลางของแผนที่ไปที่ตำแหน่งผู้ใช้

            // หาและขยาย marker ที่ใกล้ที่สุด
            highlightClosestBusStop();
        }, function() {
            alert("ไม่สามารถดึงตำแหน่งของคุณได้");
        });
    }
}

// ฟังก์ชันที่จะค้นหาจุดที่ใกล้ที่สุด และขยาย marker ของมัน
function highlightClosestBusStop() {
    if (currentLocationMarker) {
        var userLocation = currentLocationMarker.getPosition();
        var closestDistance = Infinity;
        closestMarker = null;

        // หาค่าระยะห่างจากตำแหน่งผู้ใช้ไปยังทุกจุดจอดรถ
        busMarkers.forEach(function(marker) {
            var distance = google.maps.geometry.spherical.computeDistanceBetween(userLocation, marker.getPosition());
            if (distance < closestDistance) {
                closestDistance = distance;
                closestMarker = marker;
            }
        });

        // ขยาย marker ที่ใกล้ที่สุด
        if (closestMarker) {
            var closestIcon = {
                url: 'phuket_bus-stop_green-1.png',
                scaledSize: new google.maps.Size(60, 60) // ขนาดใหญ่ขึ้น
            };
            closestMarker.setIcon(closestIcon); // ขยาย marker ที่ใกล้ที่สุด

            // ตั้งค่าขนาดของ marker อื่น ๆ กลับเป็นขนาดปกติ
            busMarkers.forEach(function(marker) {
                if (marker !== closestMarker) {
                    var defaultIcon = {
                        url: 'phuket_bus-stop_green-1.png',
                        scaledSize: new google.maps.Size(25, 25) // ขนาดปกติ
                    };
                    marker.setIcon(defaultIcon);
                }
            });
        }
    }
}

        window.onload = initMap;
    </script>
</body>
</html>
