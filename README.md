# Automatické synchronizování Garde

Jedná se o script, který po spuštění:
- Najde google kalendář GARDE a případně jej vytvoří
- Vloží do něj události ze stránky s popisem

  
## Návod
  - Otevře [Google Scripts](https://script.google.com/home)
  - Vytvořte nový projekt a obsah soubory "Kód.gs" nahraďte kódem níže
  - Na prvním řádku vložte do uvozovek mail, kam se má posílat upozornění
  - Vytvořte spouštěč (v levém menu časomíra)
      - Funkce ke spuštění - main
      - Nastavte si jak často, doporučuji "Počítadlo hodin" a nějaký čas v noci
    ![image](https://github.com/user-attachments/assets/b5ad6f1c-9879-478c-bab0-267121cf8e62)


Pro prvotní spuštění lze spustit přímo v editoru v souboru.
![image](https://github.com/user-attachments/assets/8b6b8c7e-d980-4371-aa4b-e789d1888234)


```js
const MAIL = "TVŮJ_MAIL"


String.prototype.decodeHTML = function() {
  var map = {"gt":">" /* , … */};
  return this.replace(/&(#(?:x[0-9a-f]+|\d+)|[a-z]+);?/gi, function($0, $1) {
      if ($1[0] === "#") {
          return String.fromCharCode($1[1].toLowerCase() === "x" ? parseInt($1.substr(2), 16)  : parseInt($1.substr(1), 10));
      } else {
          return map.hasOwnProperty($1) ? map[$1] : $0;
      }
  });
};

function main() {

  const theUrl = "http://garde.xf.cz/kdy-a-kde.html";
  var response = UrlFetchApp.fetch(theUrl);
  var textHtml = response.getContentText("UTF-8").decodeHTML();
  // Remove spaces
  textHtml = textHtml.replace( /\s\s+/g, ' ' );
  // Find <table
  var startI = textHtml.search("<table(.*)>");
  textHtml = textHtml.substring(startI);
  var startI = textHtml.search(">");
  textHtml = textHtml.substring(startI+1);  
  // Find </table>
  const endOfTable = "</table>";
  startI = textHtml.search(endOfTable);
  textHtml = textHtml.substring(0, startI);
  // Remove unnecesary
  textHtml = textHtml.replace(/class="(.*?)"/g, "");
  textHtml = textHtml.replace(/style="(.*?)"/g, "");
  textHtml = textHtml.replace(/<span(.*?)>/g, "");
  textHtml = textHtml.replace(/<\/span(.*?)>/g, "");
  textHtml = textHtml.replace(/<p(.*?)>/g, "");
  textHtml = textHtml.replace(/<\/p(.*?)>/g, "");
  textHtml = textHtml.replace(/\s+(?=[^<\>]*>)/g, "");
  textHtml = textHtml.replace(/\s+(?=[^<\>]*>)/g, "");

  // Find rows
  rowsHtmlText = textHtml.match(/<tr>(.*?)<\/tr>/g).map(row=>{
      const temp = row.replace(/<tr>/g,"");
      return temp.replace(/<\/tr>/g,"");
      });
  // Dump first rows
  rowsHtmlText.splice(0,3);
  // Rows to arrays 
  const rows = rowsHtmlText.map(row=>{
    const items = row.match(/<td>(.*?)<\/td>/g).map(row=>{
      const temp = row.replace(/<td>/g,"");
      return temp.replace(/<\/td>/g,"")
      });
    const dateArr = items[0].split('.');
    const day = dateArr[0];
    const month = dateArr[1];
    const year = dateArr[2];
    // console.log({str: items[0], obj: new Date(`${year}-${month}-${day}`), old: new Date(year,month-1,day)})
    return {
      // date: new Date(year,month,day),
      date: `${year}-${month.padStart(2,"0")}-${day.padStart(2,"0")}`,
      place: items[1],
      building: items[2],
      action: items[3],
      note: items[4],
    }
  })
  
  let calendarItem = Calendar.CalendarList.list().items.find(i=>i.summary === "GARDE");
  if (!calendarItem) {
    const calendar = {
      summary: "GARDE"
    };
    calendarItem = Calendar.Calendars.insert(calendar);
  }
  const existingEvents = Calendar.Events.list(calendarItem.id).items;

  const newEvents = [];
  rows.filter(r=>{
    // Only upcoming
    const now = new Date();
    const date = new Date(r.date);
    return now.getTime() < date.getTime();
  }).forEach((r,i)=>{
    // Check for existence
    var isNew = false;
    var event = existingEvents.find(e=>{      
      return e.start.date === r.date || e.start.dateTime?.split('T')[0] === r.date;
    });
    var newEvent;
    const cal = CalendarApp.getCalendarById(calendarItem.id)
    if (event === undefined) {
    // Create new event
      console.log("Event doesn't exist, creating new.", r);
      newEvent = cal.createAllDayEvent("GARDE", new Date(r.date));
      isNew = true;
    } else {
      newEvent = cal.getEventById(event.id);
    }
    // Edit description
    newEvent.setDescription(`<p><strong>Město:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.place}<br /><strong>M&iacute;sto:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.building}<br /><strong>Akce:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.action}<br /><strong>Pozn&aacute;mka:</strong> ${r.note}</p>`);
    if (isNew) newEvents.push(r);
  });
    
  if (newEvents.length === 0) return;
  
  const mailBody = newEvents.reverse().reduce((body,r,i)=>{
    return body + `<h2 style="font-size: 2rem;display:block;margin-top: 8px; margin-bottom: 8px; width: 30%;min-width: fit-content; padding-bottom: 8px; border-bottom: 1px solid WhiteSmoke;">${new Date(r.date).toLocaleDateString('cs-CZ')}</h2><p><strong>Město:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.place}<br /><strong>M&iacute;sto:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.building}<br /><strong>Akce:</strong> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${r.action}<br /><strong>Pozn&aacute;mka:</strong> ${r.note}</p>`;
    },`<h1 style="text-align: center; width: 100%; max-width: 580px; min-width: fit-content; padding-bottom:10px; border-bottom: 2px solid black; font-family:'Papyrus'; font-weight: bold; font-size: 4rem; color: Maroon ">New events</h1>`);

  // Sent notification
  MailApp.sendEmail({
  to: MAIL,
  subject: "GARDE calendar update",
  htmlBody: mailBody
  })
}

```








``
