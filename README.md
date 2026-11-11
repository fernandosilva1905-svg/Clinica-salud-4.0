//# Clinica-salud-4.0
//erronka 2
// ============================================================================
// MAQUINA DE ESTADOS AMR + CARRUSEL + TM  (adaptada al ladder del proyecto)
// ----------------------------------------------------------------------------
// El ladder mueve los equipos segun el valor de "etapa" (el Execute se mantiene
// a TRUE mientras dura la etapa, con 300 ms de retardo):
//     etapa 10, 60, 100 -> AMR  (ARCL_Goto, destino = pos_amr)       -> done_amr
//     etapa 30, 80      -> Carrusel (servo, destino = pos_carrusel)  -> done_carrusel
//     etapa 20, 40      -> Trampilla (servo, destino = pos_trampilla)-> done_trampilla
//     etapa 25          -> Lector RFID (Rfid01)                      -> doneRFID
//
// Un servo solo hace un movimiento nuevo cuando su Execute pasa de FALSE a TRUE.
// Por eso cada movimiento del carrusel es UNA VISITA NUEVA a la etapa 30 o 80:
// entre un movimiento y el siguiente el programa pasa por una etapa sin motor
// (28 / 32 / 75), el Execute baja y vuelve a subir.
//
// VARIABLES: fase_reparto (BOOL) es LOCAL nueva del programa "main".
// StartMission_Manual y Sin_Pedidos (BOOL) pasan a ser GLOBALES (VAR_EXTERNAL aqui)
// para que la HMI pueda iniciar la mision y mostrar el aviso. El resto ya existia.
//
// Etapas:
//   0 espera orden          10 AMR a Doctor          20 abre trampilla
//  25 RFID (doctor / hab.)  28 elegir ranura         30 mueve carrusel
//  32 espera al doctor      50 elegir entrega        60 AMR a habitacion
//  75 handshake TM          76 espera fin TM         80 carrusel a la ranura
//  90 espera al paciente   100 AMR a base
// ============================================================================

// LA TRAMPILLA SIEMPRE QUEDA ABIERTA:
// posicion fija a 0.0 y nunca se manda cerrar (la etapa 40 no se usa).
pos_trampilla := 0.0;
trampilla := FALSE;
trampilla1 := FALSE;

CASE etapa OF

    // ==========================================
    // 0: ESPERA DE NUEVA ORDEN (MANUAL)
    // ==========================================
    0:
        PLC_Listo := TRUE;
        activar := TRUE;          // alimenta ARCL, servos y comunicacion con el TM
        fase_reparto := FALSE;

        IF StartMission_Manual THEN

            // Contar pedidos validos (habitacion 1..5)
            total_cargas_pendientes := 0;
            FOR i := 0 TO 9 DO
                IF habitaciones[i] >= 1 AND habitaciones[i] <= 5 THEN
                    total_cargas_pendientes := total_cargas_pendientes + 1;
                END_IF;
            END_FOR;

            StartMission_Manual := FALSE;

            IF total_cargas_pendientes = 0 THEN
                Sin_Pedidos := TRUE;          // sin pedidos: no arranca
            ELSE
                Sin_Pedidos := FALSE;
                PLC_Listo := FALSE;

                FOR i := 0 TO 9 DO
                    IF habitaciones[i] >= 1 AND habitaciones[i] <= 5 THEN
                        Carrusel[i].RoomID := habitaciones[i];
                    ELSE
                        Carrusel[i].RoomID := 0;
                    END_IF;
                    Carrusel[i].MedID := medicamentos[i];
                    Carrusel[i].IsLoaded := FALSE;
                    Carrusel[i].IsDelivered := FALSE;
                END_FOR;

                cargas_realizadas := 0;
                CurrentSlot := 0;
                etapa := 10;
            END_IF;
        END_IF;

    // ==========================================
    // FASE DE CARGA EN CONSULTORIO (DOCTOR)
    // ==========================================
    10: // AMR VA AL CONSULTORIO DEL DOCTOR
        pos_amr := 'Doctor';
        IF done_amr THEN
            etapa := 20;
        END_IF;

    20: // ABRIR TRAMPILLA (se queda abierta toda la mision)
        IF done_trampilla THEN
            boton_recogido_HMI := FALSE;
            etapa := 25;
        END_IF;

    25: // RFID: lectura del doctor (fase de carga) o de la puerta (fase de reparto)
        var1 := WORD_TO_INT(In := DataFromRFID.RfidRespData[0]);

        IF NOT fase_reparto THEN
            IF doneRFID OR (var1 <> 0 AND var1 = DoctorRFID_ID) THEN
                CurrentSlot := 0;
                etapa := 28;
            END_IF;
        ELSE
            IF doneRFID OR (var1 <> 0 AND var1 = TargetRoom) THEN
                etapa := 75;
            END_IF;
        END_IF;

    28: // ELEGIR LA SIGUIENTE RANURA QUE HAY QUE CARGAR (sin motores)
        IF CurrentSlot <= 9 THEN
            IF Carrusel[CurrentSlot].RoomID > 0 AND NOT Carrusel[CurrentSlot].IsLoaded THEN

                MedID_Actual := Carrusel[CurrentSlot].MedID;
                Habitacion_Actual := Carrusel[CurrentSlot].RoomID;

                CASE MedID_Actual OF
                    1: Nombre_Medicamento := 'Paracetamol';
                    2: Nombre_Medicamento := 'Ibuprofeno';
                    3: Nombre_Medicamento := 'Antibiotico';
                    4: Nombre_Medicamento := 'Suero';
                    5: Nombre_Medicamento := 'Insulina';
                ELSE
                    Nombre_Medicamento := 'Desconocido';
                END_CASE;

                etapa := 30;
            ELSE
                CurrentSlot := CurrentSlot + 1;
            END_IF;
        ELSE
            // Todas las ranuras cargadas: empieza el reparto (trampilla sigue abierta)
            CurrentSlot := 0;
            fase_reparto := TRUE;
            etapa := 50;
        END_IF;

    30: // CARRUSEL A LA RANURA (el ladder mueve el servo en esta etapa)
        pos_carrusel := CurrentSlot * 36.0;
        IF done_carrusel THEN
            boton_recogido_HMI := FALSE;      // descarta pulsaciones anteriores
            etapa := 32;
        END_IF;

    32: // ESPERA AL DOCTOR (pulsador HMI)
        IF boton_recogido_HMI THEN
            Carrusel[CurrentSlot].IsLoaded := TRUE;
            cargas_realizadas := cargas_realizadas + 1;
            boton_recogido_HMI := FALSE;
            CurrentSlot := CurrentSlot + 1;
            etapa := 28;
        END_IF;

    // ==========================================
    // FASE DE REPARTO MULTI-HABITACION
    // ==========================================
    50: // BUSCAR LA SIGUIENTE ENTREGA PENDIENTE
        TargetRoom := 0;

        FOR i := 0 TO 9 DO
            IF Carrusel[i].IsLoaded AND NOT Carrusel[i].IsDelivered THEN
                TargetRoom := Carrusel[i].RoomID;
                CurrentSlot := i;
                EXIT;
            END_IF;
        END_FOR;

        IF TargetRoom > 0 THEN
            etapa := 60;
        ELSE
            etapa := 100;         // mision completa
        END_IF;

    60: // AMR VIAJA A LA HABITACION
        CASE Carrusel[CurrentSlot].RoomID OF
            1: pos_amr := 'habitacion1';
            2: pos_amr := 'habitacion2';
            3: pos_amr := 'habitacion3';
            4: pos_amr := 'habitacion4';
            5: pos_amr := 'habitacion5';
        END_CASE;

        IF done_amr THEN
            etapa := 25;          // lectura RFID de la puerta (fase_reparto = TRUE)
        END_IF;

    75: // HANDSHAKE ROBOT TM (24 -> 22)
        StructToTM.TM_4_O2T_RegisterInt.O2T_Register_Int[0] := 24;

        IF StructFromTM.TM_9_T2O_RegisterInt.T2O_Register_Int[0] = 22 THEN

            IF Carrusel[CurrentSlot].MedID >= 1 AND Carrusel[CurrentSlot].MedID <= 5 THEN
                IF stock_medicamentos[Carrusel[CurrentSlot].MedID] > 0 THEN
                    stock_medicamentos[Carrusel[CurrentSlot].MedID] := stock_medicamentos[Carrusel[CurrentSlot].MedID] - 1;
                END_IF;
            END_IF;

            StructToTM.TM_4_O2T_RegisterInt.O2T_Register_Int[0] := 0;
            i := 0;
            etapa := 76;
        END_IF;

    76: // ESPERAR A QUE EL TM SUELTE EL 22 (si no, la siguiente entrega se daria por confirmada)
        i := i + 1;
        IF StructFromTM.TM_9_T2O_RegisterInt.T2O_Register_Int[0] <> 22 OR i >= 500 THEN
            etapa := 80;
        END_IF;
    80: // CARRUSEL A LA RANURA DE ESTA ENTREGA (el ladder mueve el servo; trampilla ya abierta)
        pos_carrusel := CurrentSlot * 36.0;
        IF done_carrusel THEN
            boton_recogido_HMI := FALSE;
            etapa := 90;
        END_IF;
    90: // ENTREGA AL PACIENTE (trampilla abierta)
        IF boton_recogido_HMI THEN
            Carrusel[CurrentSlot].IsDelivered := TRUE;
            boton_recogido_HMI := FALSE;
            etapa := 50;          // siguiente entrega
        END_IF;

    // ==========================================
    // FASE DE RETORNO
    // ==========================================
    100: // VOLVER A BASE
        pos_amr := 'base';
        IF done_amr THEN
            etapa := 0;
        END_IF;
END_CASE;
