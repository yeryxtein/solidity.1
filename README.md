# solidity.1
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract SubastaFlash {
    address public owner;
    address public mejorOferente;
    uint public mejorOferta;

    uint public inicio;
    uint public duracion = 2 minutes;
    bool public finalizada;

    mapping(address => uint) public depositos;
    mapping(address => uint[]) public historialOfertas;
    address[] public oferentes;
    
    event NuevaOferta(address indexed oferente, uint monto);
    event SubastaFinalizada(address ganador, uint monto);
    event FondosRetirados(address indexed to, uint amount);
    event ReembolsoParcial(address indexed usuario, uint monto);

    constructor() {
        owner = msg.sender;
        inicio = block.timestamp;
    }

    modifier subastaActiva() {
        require(!finalizada, "Subasta finalizada");
        require(block.timestamp <= inicio + duracion, "Tiempo terminado");
        _;
    }

    modifier soloOwner() {
        require(msg.sender == owner, "No eres el owner");
        _;
    }

    function ofertar() external payable subastaActiva {
        require(msg.value > 0, "Debes enviar ETH");

        uint total = depositos[msg.sender] + msg.value;
        uint minimoRequerido = mejorOferta + (mejorOferta * 5) / 100;

        require(total >= minimoRequerido, "La oferta debe superar en al menos 5%");

        if (depositos[msg.sender] == 0) {
            oferentes.push(msg.sender);
        }

        historialOfertas[msg.sender].push(msg.value);
        depositos[msg.sender] = total;
        mejorOferente = msg.sender;
        mejorOferta = total;

        if ((inicio + duracion) - block.timestamp <= 10 minutes) {
            duracion += 10 minutes;
        }

        emit NuevaOferta(msg.sender, total);
    }

    function solicitarReembolsoParcial() external subastaActiva {
        uint[] storage ofertas = historialOfertas[msg.sender];
        require(ofertas.length > 1, "No hay ofertas anteriores para reembolsar");

        uint montoReembolso = 0;
        for (uint i = 0; i < ofertas.length - 1; i++) {
            montoReembolso += ofertas[i];
        }

        require(montoReembolso > 0, "Nada que reembolsar");

        historialOfertas[msg.sender] = [ofertas[ofertas.length - 1]];
        depositos[msg.sender] -= montoReembolso;

        (bool success, ) = payable(msg.sender).call{value: montoReembolso}("");
        require(success, "Fallo en el reembolso parcial");

        emit ReembolsoParcial(msg.sender, montoReembolso);
    }

    function mostrarOfertas() external view returns (address[] memory, uint[] memory) {
        uint[] memory montos = new uint[](oferentes.length);
        for (uint i = 0; i < oferentes.length; i++) {
            montos[i] = depositos[oferentes[i]];
        }
        return (oferentes, montos);
    }

    function finalizar() external soloOwner {
        require(!finalizada, "Ya finalizo");
        require(block.timestamp >= inicio + duracion, "Aun no termina");

        finalizada = true;

        emit SubastaFinalizada(mejorOferente, mejorOferta);
    }

    function retirar() external {
        require(finalizada, "Subasta no finalizada");
        require(msg.sender != mejorOferente, "Ganador no puede retirar");

        uint deposito = depositos[msg.sender];
        require(deposito > 0, "Nada para retirar");

        uint comision = (deposito * 2) / 100;
        uint reembolso = deposito - comision;

        depositos[msg.sender] = 0;

        (bool exito, ) = payable(msg.sender).call{value: reembolso}("");
        require(exito, "Fallo en retiro");
    }

    function retirarFondos() external soloOwner {
        require(finalizada, "Subasta no finalizada");
        require(address(this).balance > 0, "Sin balance disponible");

        uint monto = address(this).balance;
        (bool exito, ) = payable(owner).call{value: monto}("");
        require(exito, "Fallo al transferir al owner");

        emit FondosRetirados(owner, monto);
    }

    function tiempoRestante() external view returns (uint) {
        if (block.timestamp >= inicio + duracion || finalizada) {
            return 0;
        } else {
            return (inicio + duracion) - block.timestamp;
        }
    }

    function verBalance() external view returns (uint) {
        return address(this).balance;
    }
}
