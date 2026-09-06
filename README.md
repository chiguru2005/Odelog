<!DOCTYPE html>
<html lang="ja">

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width, initial-scale=1.0">

<title>📍おでログ Ver.3.0</title>

<style>

body{
    font-family:sans-serif;
    background:#f5f5f5;
    margin:0;
    padding:20px;
}

h1{
    text-align:center;
    font-size:36px;
}

button{
    width:100%;
    font-size:22px;
    padding:17px;
    margin:8px 0;
    border:none;
    border-radius:15px;
    box-shadow:0 3px 10px rgba(0,0,0,0.2);
}

#startBtn{
    background:#4CAF50;
    color:white;
}

#stopBtn{
    background:#f44336;
    color:white;
}

#locationBtn{
    background:#2196F3;
    color:white;
}

#refreshBtn{
    background:#FF9800;
    color:white;
}

#txtBtn{
    background:#9C27B0;
    color:white;
}

#status{
    text-align:center;
    font-size:21px;
    color:green;
    margin:15px;
    min-height:30px;
}

.box{
    background:white;
    padding:15px;
    border-radius:15px;
    margin-top:20px;
}

.record{
    border-bottom:1px solid #ddd;
    padding:14px 0;
    font-size:18px;
    line-height:1.7;
}

.placeName{
    font-size:23px;
    font-weight:bold;
}

.station{
    font-size:21px;
    font-weight:bold;
}

.info{
    color:#555;
}

.small{
    color:#777;
    font-size:15px;
}

.pageButtons{
    display:flex;
    gap:10px;
}

.pageButtons button{
    font-size:19px;
}

#trackingState{
    text-align:center;
    font-size:20px;
    font-weight:bold;
}

</style>

</head>


<body>

<h1>📍おでログ</h1>


<button id="startBtn">
▶️ 自動記録開始
</button>


<button id="stopBtn">
⏹️ 自動記録停止
</button>


<button id="locationBtn">
🏠 現在地を見る
</button>


<button id="refreshBtn">
🔄 更新
</button>


<div id="status">
待機中
</div>


<div class="box">

<h2>📡 自動記録</h2>

<div id="trackingState">
停止中
</div>

<p>
自動記録中は、約1分ごとにGPSを確認します。
</p>

</div>


<div class="box">

<h2>📍 現在地</h2>

<p id="place">
場所：未取得
</p>

<p id="station">
駅：未取得
</p>

<p id="nowTime">
時間：未取得
</p>

<p id="coordinates">
GPS：未取得
</p>

</div>


<div class="box">

<h2>🗺️ 今日のタイムライン</h2>

<div id="timeline">
まだ記録はありません
</div>

</div>


<div class="box">

<h2>📋 GPS記録</h2>

<div id="list">
まだ記録はありません
</div>

</div>


<div class="pageButtons">

<button id="prevBtn">
◀ 前へ
</button>

<button id="nextBtn">
次へ ▶
</button>

</div>


<button id="txtBtn">
💾 今日の記録をTXT保存
</button>


<script>


/* ==================================================
   データ
================================================== */

let records =
JSON.parse(
    localStorage.getItem("odeLogRecords")
) || [];


let page = 0;

const perPage = 5;

let tracking = false;

let trackingTimer = null;


/* ==================================================
   HTML
================================================== */

const status =
document.getElementById("status");

const list =
document.getElementById("list");

const timeline =
document.getElementById("timeline");

const placeText =
document.getElementById("place");

const stationText =
document.getElementById("station");

const timeText =
document.getElementById("nowTime");

const coordinatesText =
document.getElementById("coordinates");

const trackingState =
document.getElementById("trackingState");


/* ==================================================
   HTML安全化
================================================== */

function escapeHTML(text){

    if(text === undefined ||
       text === null){

        return "";

    }

    return String(text)
        .replace(/&/g,"&amp;")
        .replace(/</g,"&lt;")
        .replace(/>/g,"&gt;")
        .replace(/"/g,"&quot;")
        .replace(/'/g,"&#039;");

}


/* ==================================================
   今日の日付
================================================== */

function getToday(){

    const now =
    new Date();

    return (
        now.getFullYear() +
        "-" +
        String(
            now.getMonth()+1
        ).padStart(2,"0") +
        "-" +
        String(
            now.getDate()
        ).padStart(2,"0")
    );

}


/* ==================================================
   時刻
================================================== */

function getTime(){

    const now =
    new Date();

    return (
        String(now.getHours()).padStart(2,"0") +
        ":" +
        String(now.getMinutes()).padStart(2,"0") +
        ":" +
        String(now.getSeconds()).padStart(2,"0")
    );

}


/* ==================================================
   GPS
================================================== */

function getGPS(){

    return new Promise(
        function(resolve,reject){

            if(!navigator.geolocation){

                reject(
                    new Error(
                        "GPSに対応していません"
                    )
                );

                return;

            }


            navigator.geolocation.getCurrentPosition(

                resolve,

                reject,

                {
                    enableHighAccuracy:true,
                    timeout:20000,
                    maximumAge:0
                }

            );

        }
    );

}


/* ==================================================
   距離計算
================================================== */

function getDistance(
    lat1,
    lon1,
    lat2,
    lon2
){

    const R = 6371000;

    const rad =
    Math.PI / 180;

    const dLat =
    (lat2-lat1) * rad;

    const dLon =
    (lon2-lon1) * rad;

    const a =
    Math.sin(dLat/2) *
    Math.sin(dLat/2) +

    Math.cos(lat1*rad) *
    Math.cos(lat2*rad) *

    Math.sin(dLon/2) *
    Math.sin(dLon/2);

    const c =
    2 *
    Math.atan2(
        Math.sqrt(a),
        Math.sqrt(1-a)
    );

    return R*c;

}


/* ==================================================
   地名取得
================================================== */

async function getPlaceName(lat,lon){

    try{

        const url =
        "https://nominatim.openstreetmap.org/reverse" +
        "?format=json" +
        "&lat=" + lat +
        "&lon=" + lon +
        "&accept-language=ja";

        const response =
        await fetch(url);

        const data =
        await response.json();

        const address =
        data.address || {};


        return (

            address.road ||

            address.neighbourhood ||

            address.suburb ||

            address.city ||

            address.town ||

            address.village ||

            address.municipality ||

            address.county ||

            "場所不明"

        );

    }catch(error){

        return "地名取得失敗";

    }

}


/* ==================================================
   最寄り駅
================================================== */

async function getNearestStation(lat,lon){

    try{

        const query = `
[out:json][timeout:10];

(
  node
    ["railway"="station"]
    (around:1500,${lat},${lon});

  way
    ["railway"="station"]
    (around:1500,${lat},${lon});
);

out center tags;
`;


        const response =
        await fetch(
            "https://overpass-api.de/api/interpreter",
            {
                method:"POST",
                body:query
            }
        );


        const data =
        await response.json();


        if(
            !data.elements ||
            data.elements.length===0
        ){

            return "駅情報なし";

        }


        let nearest=null;

        let nearestDistance=Infinity;


        data.elements.forEach(
            function(element){

                const stationLat =
                element.lat ??
                element.center?.lat;

                const stationLon =
                element.lon ??
                element.center?.lon;


                if(
                    stationLat === undefined ||
                    stationLon === undefined
                ){

                    return;

                }


                const distance =
                getDistance(
                    lat,
                    lon,
                    stationLat,
                    stationLon
                );


                if(
                    distance <
                    nearestDistance
                ){

                    nearestDistance =
                    distance;


                    nearest={

                        name:
                        element.tags?.name ||
                        "名称不明",

                        distance:
                        distance

                    };

                }

            }
        );


        if(!nearest){

            return "駅情報なし";

        }


        return (
            nearest.name +
            "（約" +
            Math.round(
                nearest.distance
            ) +
            "m）"
        );

    }catch(error){

        return "駅情報取得失敗";

    }

}


/* ==================================================
   滞在時間
================================================== */

function formatDuration(ms){

    if(!ms || ms<0){

        return "0秒";

    }


    const totalSeconds =
    Math.floor(ms/1000);


    const hours =
    Math.floor(
        totalSeconds/3600
    );


    const minutes =
    Math.floor(
        (totalSeconds%3600)/60
    );


    const seconds =
    totalSeconds%60;


    let result="";


    if(hours>0){

        result +=
        hours + "時間";

    }


    if(
        minutes>0 ||
        hours>0
    ){

        result +=
        minutes + "分";

    }


    result +=
    seconds + "秒";


    return result;

}


/* ==================================================
   移動判定
================================================== */

function guessMovement(
    distance,
    seconds
){

    if(seconds<=0){

        return "不明";

    }


    const speed =
    distance / seconds;


    /*
       m/s

       0〜1.8
       徒歩・車椅子など

       1.8〜8
       自転車・低速車両など

       8以上
       電車・バス・車など
    */


    if(speed < 1.8){

        return "🚶 ゆっくり移動";

    }


    if(speed < 8){

        return "🚲 速い移動";

    }


    return "🚌🚃🚗 乗り物による移動の可能性";

}


/* ==================================================
   記録
================================================== */

async function recordPosition(){

    if(!tracking){

        return;

    }


    try{

        status.innerHTML =
        "📡 GPSを確認しています...";


        const position =
        await getGPS();


        const lat =
        position.coords.latitude;

        const lon =
        position.coords.longitude;


        status.innerHTML =
        "🌍 場所を調べています...";


        const place =
        await getPlaceName(
            lat,
            lon
        );


        status.innerHTML =
        "🚉 駅を調べています...";


        const station =
        await getNearestStation(
            lat,
            lon
        );


        const now =
        new Date();


        const timestamp =
        now.getTime();


        const date =
        getToday();


        const time =
        getTime();


        let distance=0;

        let moveTime=0;

        let movement="";


        const previous =
        records.length>0
        ? records[0]
        : null;


        if(previous){

            distance =
            getDistance(
                Number(previous.latitude),
                Number(previous.longitude),
                lat,
                lon
            );


            moveTime =
            timestamp -
            Number(previous.timestamp);


            movement =
            guessMovement(
                distance,
                moveTime/1000
            );

        }


        const data={

            place:place,

            station:station,

            time:time,

            date:date,

            timestamp:timestamp,

            latitude:lat,

            longitude:lon,

            distanceFromPrevious:
            distance,

            moveTime:
            moveTime,

            movement:
            movement

        };


        records.unshift(data);


        localStorage.setItem(
            "odeLogRecords",
            JSON.stringify(records)
        );


        updateCurrent(
            data
        );


        page=0;

        showRecords();

        showTimeline();


        status.innerHTML =
        "✅ 自動記録：" +
        time;


    }catch(error){

        console.log(error);

        status.innerHTML =
        "❌ GPS取得に失敗しました";

    }

}


/* ==================================================
   現在地表示
================================================== */

function updateCurrent(data){

    placeText.innerHTML =
    "場所：" +
    escapeHTML(data.place);


    stationText.innerHTML =
    "駅：" +
    escapeHTML(data.station);


    timeText.innerHTML =
    "時間：" +
    escapeHTML(data.time);


    coordinatesText.innerHTML =
    "GPS：" +
    Number(data.latitude).toFixed(6) +
    " / " +
    Number(data.longitude).toFixed(6);

}


/* ==================================================
   自動記録開始
================================================== */

document
.getElementById("startBtn")
.onclick=function(){

    if(tracking){

        status.innerHTML =
        "すでに自動記録中です";

        return;

    }


    tracking=true;


    trackingState.innerHTML =
    "🟢 自動記録中";


    status.innerHTML =
    "▶️ 自動記録を開始しました";


    /*
       まずすぐ1回
    */

    recordPosition();


    /*
       約1分ごと
    */

    trackingTimer =
    setInterval(
        recordPosition,
        60000
    );

};


/* ==================================================
   自動記録停止
================================================== */

document
.getElementById("stopBtn")
.onclick=function(){

    tracking=false;


    if(trackingTimer){

        clearInterval(
            trackingTimer
        );

        trackingTimer=null;

    }


    trackingState.innerHTML =
    "🔴 停止中";


    status.innerHTML =
    "⏹️ 自動記録を停止しました";

};


/* ==================================================
   現在地を見る
================================================== */

document
.getElementById("locationBtn")
.onclick=async function(){

    try{

        status.innerHTML =
        "📡 現在地を取得中...";


        const position =
        await getGPS();


        const lat =
        position.coords.latitude;

        const lon =
        position.coords.longitude;


        const place =
        await getPlaceName(
            lat,
            lon
        );


        const station =
        await getNearestStation(
            lat,
            lon
        );


        const data={

            place:place,

            station:station,

            latitude:lat,

            longitude:lon,

            time:getTime()

        };


        updateCurrent(data);


        status.innerHTML =
        "📍 現在地を表示しました";


    }catch(error){

        status.innerHTML =
        "❌ 現在地を取得できませんでした";

    }

};


/* ==================================================
   今日の記録
================================================== */

function getTodayRecords(){

    const today =
    getToday();


    return records
        .filter(
            function(r){

                return r.date===today;

            }
        )
        .sort(
            function(a,b){

                return (
                    Number(a.timestamp) -
                    Number(b.timestamp)
                );

            }
        );

}


/* ==================================================
   タイムライン
================================================== */

function showTimeline(){

    const todayRecords =
    getTodayRecords();


    timeline.innerHTML="";


    if(
        todayRecords.length===0
    ){

        timeline.innerHTML =
        "今日の記録はありません";

        return;

    }


    todayRecords.forEach(
        function(r,index){

            const div =
            document.createElement(
                "div"
            );


            div.className="record";


            let extra="";


            if(index>0){

                const previous =
                todayRecords[index-1];


                const duration =
                Number(r.timestamp) -
                Number(previous.timestamp);


                const distance =
                getDistance(
                    Number(previous.latitude),
                    Number(previous.longitude),
                    Number(r.latitude),
                    Number(r.longitude)
                );


                const movement =
                guessMovement(
                    distance,
                    duration/1000
                );


                extra =

                "<br>" +

                "↳ 前地点から：" +
                formatDuration(
                    duration
                ) +

                "<br>" +

                "↳ 移動距離：約" +
                Math.round(distance) +
                "m" +

                "<br>" +

                "↳ " +
                movement;

            }


            div.innerHTML =

            "<div class='placeName'>" +

            "📍 " +
            escapeHTML(r.place) +

            "</div>" +

            "🕒 " +
            escapeHTML(r.time) +

            "<br>" +

            "🚉 " +
            escapeHTML(r.station) +

            extra +

            "<br>" +

            "<span class='small'>" +

            "GPS：" +

            Number(r.latitude).toFixed(6) +

            " / " +

            Number(r.longitude).toFixed(6) +

            "</span>";


            timeline.appendChild(
                div
            );

        }
    );

}


/* ==================================================
   履歴
================================================== */

function showRecords(){

    list.innerHTML="";


    const start =
    page * perPage;


    const show =
    records.slice(
        start,
        start+perPage
    );


    if(show.length===0){

        list.innerHTML =
        "まだ記録はありません";

        return;

    }


    show.forEach(
        function(r){

            const div =
            document.createElement(
                "div"
            );


            div.className="record";


            div.innerHTML =

            "<div class='station'>" +

            "🚉 " +

            escapeHTML(
                r.station
            ) +

            "</div>" +

            "📍 " +

            escapeHTML(
                r.place
            ) +

            "<br>" +

            "🕒 " +

            escapeHTML(
                r.time
            ) +

            "<br>" +

            "<span class='info'>" +

            "移動：" +

            escapeHTML(
                r.movement ||
                "最初の記録"
            ) +

            "</span>" +

            "<br>" +

            "<span class='small'>" +

            "GPS：" +

            Number(
                r.latitude
            ).toFixed(6) +

            " / " +

            Number(
                r.longitude
            ).toFixed(6) +

            "</span>";


            list.appendChild(
                div
            );

        }
    );

}


/* ==================================================
   更新
================================================== */

document
.getElementById("refreshBtn")
.onclick=function(){

    records =
    JSON.parse(
        localStorage.getItem(
            "odeLogRecords"
        )
    ) || [];


    page=0;


    showRecords();

    showTimeline();


    status.innerHTML =
    "🔄 更新しました";

};


/* ==================================================
   前へ
================================================== */

document
.getElementById("prevBtn")
.onclick=function(){

    if(page>0){

        page--;

        showRecords();

    }

};


/* ==================================================
   次へ
================================================== */

document
.getElementById("nextBtn")
.onclick=function(){

    if(
        (page+1)*perPage
        <
        records.length
    ){

        page++;

        showRecords();

    }

};


/* ==================================================
   TXT作成
================================================== */

function makeTXT(){

    const todayRecords =
    getTodayRecords();


    let text =
    "📍 おでログ Ver.3.0\n\n";


    text +=
    "日付：" +
    getToday() +
    "\n";


    text +=
    "================================\n\n";


    if(
        todayRecords.length===0
    ){

        text +=
        "本日の記録はありません。\n";

        return text;

    }


    todayRecords.forEach(
        function(r,index){

            text +=
            "【" +
            (index+1) +
            "】\n";


            text +=
            "時刻：" +
            r.time +
            "\n";


            text +=
            "場所：" +
            r.place +
            "\n";


            text +=
            "駅：" +
            r.station +
            "\n";


            if(index>0){

                const previous =
                todayRecords[index-1];


                const duration =
                Number(r.timestamp) -
                Number(previous.timestamp);


                const distance =
                getDistance(
                    Number(previous.latitude),
                    Number(previous.longitude),
                    Number(r.latitude),
                    Number(r.longitude)
                );


                const movement =
                guessMovement(
                    distance,
                    duration/1000
                );


                text +=
                "前地点からの時間：" +
                formatDuration(
                    duration
                ) +
                "\n";


                text +=
                "移動距離：約" +
                Math.round(
                    distance
                ) +
                "m\n";


                text +=
                "移動判定：" +
                movement +
                "\n";

            }


            text +=
            "緯度：" +
            r.latitude +
            "\n";


            text +=
            "経度：" +
            r.longitude +
            "\n";


            text +=
            "--------------------------------\n\n";

        }
    );


    return text;

}


/* ==================================================
   TXT保存
================================================== */

document
.getElementById("txtBtn")
.onclick=function(){

    const filename =
    getToday() +
    "_おでログ.txt";


    const text =
    makeTXT();


    const blob =
    new Blob(
        [text],
        {
            type:
            "text/plain;charset=utf-8"
        }
    );


    const url =
    URL.createObjectURL(
        blob
    );


    const a =
    document.createElement(
        "a"
    );


    a.href=url;

    a.download=filename;


    document.body.appendChild(a);

    a.click();

    document.body.removeChild(a);


    URL.revokeObjectURL(url);


    status.innerHTML =
    "💾 今日のTXTを作成しました";

};


/* ==================================================
   起動
================================================== */

showRecords();

showTimeline();


</script>

</body>

</html>
