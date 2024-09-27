# Supported Blockchains and Tokens 

We list below the different bridgable tokens with XLink. 

{% hint style="info" %}
**Important:** Users can only bridge tokens that represent the **same asset** across different blockchains. For example, BTC can be transferred to its equivalent, WBTC, when moving from Bitcoin to an EVM network, as both represent the same asset on different chains.
{% endhint %}

| **Blockchain** | **Tokens**                                                                                                        |
|----------------|-------------------------------------------------------------------------------------------------------------------|
| **AILayer**    | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Arbitrum**   | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Aurora**     | aBTC, ALEX, sUSDT, vLiALEX, vLiSTX                                                                                 |
| **B²**         | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Bitcoin**    | BTC                                                                                                                |
| **Bitlayer**   | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **BNB**        | aBTC, ALEX, BTCB, sUSDT, SKO, USDT, uBTC                                                                            |
| **BOB**        | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Core**       | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Ethereum**   | ALEX, sUSDT, WBTC                                                                                                  |
| **Lorenzo**    | aBTC, ALEX, sUSDT, vLiALEX, vLiSTX                                                                                 |
| **Merlin**     | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Mode**       | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |
| **Runes**      | DOG•GO•TO•THE•MOON, ETHEREUM•ON•BITCOIN, MAKE•BITCORN•GREAT•AGAIN, NOT•GONNA•MAKE•IT, SATOSHI•NAKAMOTO•INU, WELSH•CORGI•COIN |
| **Stacks**     | aBTC, ALEX, DOG•GO•TO•THE•MOON, ETHEREUM, NOT, PEPE, SATOSHI•NAKAMOTO•INU, SKO, sUSDT, uBTC, vLiALEX, vLiSTX, WELSH  |
| **X Layer**    | aBTC, ALEX, sUSDT, uBTC, vLiALEX, vLiSTX                                                                           |


# Preview Test

<div>
  <label for="sourceBlockchain">Source Blockchain:</label>
  <select id="sourceBlockchain" onchange="updateTable()">
    <option value="BTC">BTC</option>
    <option value="BNB">BNB</option>
  </select>

  <label for="destinationBlockchain">Destination Blockchain:</label>
  <select id="destinationBlockchain" onchange="updateTable()">
    <option value="BNB">BNB</option>
    <option value="BTC">BTC</option>
  </select>
</div>

<table id="tokenTable">
  <thead>
    <tr>
      <th id="sourceHeader">Source Token</th>
      <th id="destinationHeader">Destination Token</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Select a source and destination blockchain</td>
      <td></td>
    </tr>
  </tbody>
</table>

<script>
  const tokenMappings = {
    "BTC": {
      "BNB": ["BTCB", "aBTC"],
      "BTC": ["BTC"]
    },
    "BNB": {
      "BTC": ["BTCB"],
      "BNB": ["BNB"]
    }
  };

  function updateTable() {
    const source = document.getElementById("sourceBlockchain").value;
    const destination = document.getElementById("destinationBlockchain").value;

    const tableBody = document.getElementById("tokenTable").querySelector("tbody");
    tableBody.innerHTML = "";  // Clear the table

    if (tokenMappings[source] && tokenMappings[source][destination]) {
      const mappings = tokenMappings[source][destination];
      mappings.forEach((destToken) => {
        const row = document.createElement("tr");
        const sourceCell = document.createElement("td");
        const destCell = document.createElement("td");
        sourceCell.textContent = source;
        destCell.textContent = destToken;
        row.appendChild(sourceCell);
        row.appendChild(destCell);
        tableBody.appendChild(row);
      });
    } else {
      const row = document.createElement("tr");
      const sourceCell = document.createElement("td");
      const destCell = document.createElement("td");
      sourceCell.textContent = "No mappings found";
      destCell.textContent = "";
      row.appendChild(sourceCell);
      row.appendChild(destCell);
      tableBody.appendChild(row);
    }

    // Update Headers
    document.getElementById("sourceHeader").textContent = source + " Token";
    document.getElementById("destinationHeader").textContent = destination + " Token";
  }
</script>

