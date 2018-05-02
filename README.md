from datetime import datetime, timedelta, time
from RPi import GPIO
from threading import Thread
import logging
import time
from smbus import SMBus

logging.basicConfig(level=logging.DEBUG) # , filename='output_klok.log')
log = logging.getLogger()
GPIO.setmode(GPIO.BCM)

i2c=SMBus(1)

DS1307_DEFAULT_ADDRESS = 0x68

# Registers
DS3231_REG_SECONDS = DS3231_REG_DATETIME = 0x0
DS3231_REG_MINUTES = 0x1
DS3231_REG_HOURS = 0x2
DS3231_REG_DAY = 0x3
DS3231_REG_DATE = 0x4
DS3231_REG_MONTH = 0x5
DS3231_REG_YEAR = 0x6
DS3231_REG_ALARM1_SECONDS = DS3231_REG_ALARM1 = 0x7
DS3231_REG_ALARM1_MINUTES = 0x8
DS3231_REG_ALARM1_HOURS = 0x9
DS3231_REG_ALARM1_DAY_DATE = 0xA
DS3231_REG_ALARM2_MINUTES = 0xB
DS3231_REG_ALARM2_HOURS = 0xC
DS3231_REG_ALARM2_DAY_DATE = 0xD
DS3231_REG_CONTROL = 0xE
DS3231_REG_STATUS = 0xF
DS3231_REG_TEMP_MSB = 0x11
DS3231_REG_TEMP_LSB = 0x12

led = 22
min_knop = 27
plus_knop = 12
waarde = 0
waarde_alarm = 0
alarm_seconds = 0
alarm_ingesteld = False
gekozen_cijfer = 0
aantal_drukken = 0
gekozen_modus = 1


# default pins
PIN_DISPLAY_CATHODES = (18, 24, 23, 25)
PIN_DISPLAY = (26, 19, 13, 6 ,5)

for digit in PIN_DISPLAY_CATHODES:
    GPIO.setup(digit, GPIO.OUT)
    GPIO.output(digit, 0)

# timing - pas aan indien nodig
FREQ = 1e7  # 10 MHz
PERIOD = 1 / FREQ
DELAY = PERIOD / 2  # T_low, T_high = 1/2 periode

PIN_DS3231_INT_SQW = 17
PIN_ROTENC_CLK = 21
PIN_ROTENC_DT = 20
PIN_ROTENC_SW = 16

DS = 5  # serial data
OE = 6  # output enable (active low)
STCP = 13  # storage register clock pulse
SHCP = 19  # shift register clock pulse
MR = 26  # master reset (active low)

class DS3121:
    def __init__(self, curr_date: datetime):
        self.set_datetime(curr_date)
        self.get_datetime()

    def __bcd2int(self, value):
        mask = 0x0f
        return (value >> 4) * 10 + (value & mask)


    def __int_to_bcd(self, value):
        result = ((value // 10) << 4) | (value % 10)
        return result


    # stelt de tijd in a.d.h.v een datetime-object
    def set_datetime(self, value):
        jaar = value.year
        maand = value.month
        dag = value.day
        uur = value.hour
        minuten = value.minute
        seconden = value.second
        # log.info((minuten))


        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_SECONDS, (self.__int_to_bcd(seconden)))
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_MINUTES, (self.__int_to_bcd(minuten)))
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_HOURS, (self.__int_to_bcd(uur)))
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_DAY, (self.__int_to_bcd(dag)))
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_MONTH, (self.__int_to_bcd(maand)))
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_YEAR, (self.__int_to_bcd(jaar)))


    def get_datetime(self):

            seconden = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_SECONDS))
            minuten = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_MINUTES))
            uur = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_HOURS))
            dag = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_DAY))
            maand = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_MONTH))
            jaar = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_YEAR))

            # log.info(self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_STATUS)))
            # log.info(self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_SECONDS)))
            # log.info(self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_MINUTES)))
            # log.info(self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_HOURS)))
            # log.info(self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_DAY_DATE)))

            # global alarm_seconds
            # alarm_seconds = self.__bcd2int(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_SECONDS))
            return datetime(2018,maand,28,uur,minuten,seconden)


    def set_timer(self,value2):
        opties_register = [DS3231_REG_ALARM1_SECONDS, DS3231_REG_ALARM1_MINUTES, DS3231_REG_ALARM1_HOURS]
        global gekozen_cijfer
        global vorige_status_knop
        global alarm_ingesteld
        global aantal_drukken
        gekozen_cijfer = 0
        aantal_drukken += 1
        #het alarm terug op 0 instellen
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_MINUTES, 0)
        if aantal_drukken == 1:
            log.info("Het alarm kan worden ingesteld")
            # log.info("Druk op de knop om te veranderen tussen seconden, minuten en uren")
        elif aantal_drukken == 2:
            log.info("U kunt nu instellen na hoveel seconden het alarm zal afgaan")
            log.info("Druk nog eens op de rotary encoder om het alarm te bevestigen")
        elif aantal_drukken == 3:
            now = rtc.get_datetime()
            # opties = [now.second, now.minute, now.hour]
            instellen_op = now + timedelta(seconds=5)
            log.info(now.second)
            log.info(instellen_op.second)
            # i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, opties_register[gekozen_cijfer],
            #                     int_to_bcd((instellen_op.second)))
            # i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_MINUTES, (int_to_bcd(instellen_op.minute)))
            # i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_HOURS, (int_to_bcd(instellen_op.hour)))
            # i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_DAY_DATE, (int_to_bcd(instellen_op.day)))
            i2c.write_block_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_ALARM1_SECONDS, [instellen_op.second, instellen_op.minute, instellen_op.hour, instellen_op.day])
            alarm_ingesteld = True
            log.info("Het alarm werd bevestigd")
        else:
            aantal_drukken = 0
            for optie in opties_register:
                i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, optie, 0)


class FourSevenSegment:
    def __init__(self, ds_pin=DS, shcp_pin=SHCP, stcp_pin=STCP, mr_pin=MR, oe_pin=OE):
        self.ds_pin = ds_pin
        self.shcp_pin = shcp_pin
        self.stcp_pin = stcp_pin
        self.mr_pin = mr_pin
        self.oe_pin = oe_pin

        GPIO.setup(DS, GPIO.OUT)
        GPIO.setup(SHCP, GPIO.OUT)
        GPIO.setup(STCP, GPIO.OUT)
        GPIO.setup(MR, GPIO.OUT)
        GPIO.setup(OE, GPIO.OUT)

        GPIO.output(DS, 0)
        GPIO.output(SHCP, 0)
        GPIO.output(STCP, 0)
        GPIO.output(OE, 0)
        GPIO.output(MR, 1)


    def write_bit(self, value):

        GPIO.output(DS, value) #bit klaar zetten (value)
        GPIO.output(SHCP, 0)  # rising edge - we gaan ervan uit dat de pin altijd LOW wordt achtergelaten!
        time.sleep(DELAY)  # halve klokcyclus wachten
        GPIO.output(SHCP, 1)  # falling edge - zorg dat de pin altijd weer LOW eindigt
        time.sleep(DELAY)

    def copy_to_storage_register(self):

        GPIO.output(STCP, 0)  # rising edge - we gaan ervan uit dat de pin altijd LOW wordt achtergelaten!
        time.sleep(DELAY)  # halve klokcyclus wachten
        GPIO.output(STCP, 1)  # falling edge - zorg dat de pin altijd weer LOW eindigt
        time.sleep(DELAY)  # halve klokcyclus wachten

    def write_byte(self, value):
        mask = 0x80 # de linkse bit = 0
        while mask :
            self.write_bit(value & mask) # overlopen van msb naar lsb (least significant bit)
            mask=mask >> 1
        self.copy_to_storage_register()

# Juiste bit voor elk segment: vul/pas aan
A = 1 << 0  # 0000 0001
B = 1 << 1  # 0000 0010
C = 1 << 2
D = 1 << 3
E = 1 << 4
F = 1 << 5
G = 1 << 6
DP = 1 << 7

# Juist segment voor elk (hex) cijfer: vul/pas aan
SEGMENTS = {
    0x0: G | DP,
    0x1: A | F | E | D | G | DP,
    0x2: F | C | DP,
    0x3: E | F | DP,
    0x4: A | E | D | DP,
    0x5: B | E | DP,
    0x6: B | DP,
    0x7: F | G | E | D | DP,
    0x8: DP,
    0x9: E | DP
}

SEGMENTS_2 = {
    0x0: G ,
    0x1: A | F | E | D | G,
    0x2: F | C ,
    0x3: E | F ,
    0x4: A | E | D ,
    0x5: B | E ,
    0x6: B ,
    0x7: F | G | E | D ,
    0x8: DP,
    0x9: E
}

class set_value:
    # toont een getal tussen 0-9999
    def __init__(self, shreg: FourSevenSegment):
        self.shreg = shreg

    def show_segments(self, value):
        self.shreg.write_byte(value)

    def show_digit(self, value, with_dp=False):
        char = SEGMENTS[value]
        self.show_segments(char)


    def show_digit_2(self, value, with_dp=False):
        char = SEGMENTS_2[value]
        self.show_segments(char)


def lights():
    for digit in PIN_DISPLAY_CATHODES:
        GPIO.output(digit, 0)


def veranderen_van(ch=None):
    global gekozen_cijfer
    if gekozen_cijfer == 0:
        log.info("U koos om de seconden te veranderen")
    elif gekozen_cijfer == 1:
        log.info("U koos om de minuten te veranderen")
    elif gekozen_cijfer == 2:
        log.info("U koos om het uur te veranderen")
    else:
        gekozen_cijfer = 0


def waarde_plus(ch=None):
    if GPIO.event_detected(plus_knop):
        time.sleep(0.5)
        global waarde
        global gekozen_cijfer
        global aantal_drukken
        global gekozen_modus
        if aantal_drukken % 2 != 0:
            gekozen_cijfer += 1
            veranderen_van()
        elif aantal_drukken !=0 or gekozen_cijfer >=2:
            waarde += 1
            value()
        elif aantal_drukken == 0:
            gekozen_modus += 1
            modus()
            log.info(datetime(DS3121(datetime.now()).get_datetime().year,DS3121(datetime.now()).get_datetime().month,DS3121(datetime.now()).get_datetime().day, DS3121(datetime.now()).get_datetime().hour, DS3121(datetime.now()).get_datetime().minute,DS3121(datetime.now()).get_datetime().second))
    return waarde


def waarde_min(ch=None):
    if GPIO.event_detected(min_knop):
        global waarde
        global gekozen_cijfer
        global aantal_drukken
        global gekozen_modus
        if aantal_drukken % 2 != 0:
            gekozen_cijfer -= 1
            if gekozen_cijfer <= 0 or gekozen_cijfer >=2:
                gekozen_cijfer = 0
            veranderen_van()
        elif aantal_drukken !=0 or gekozen_cijfer >=2:
            waarde -= 1
            value()
            veranderen_van()
        elif aantal_drukken == 0:
            gekozen_modus -= 1
            modus()
        if waarde < 0 :
            waarde = 0
    return waarde


def modus(ch=None):
    global gekozen_modus
    if gekozen_modus == 1:
        log.info("De modus werd veranderd in minuten & seconden")
    elif gekozen_modus == 2:
        log.info("De modus werd veranderd in uren & minuten")
    elif gekozen_modus == 3:
        log.info("De modus werd veranderd in maand & dag")
    else:
        gekozen_modus = 0


def value():
    global waarde_alarm
    waarde_alarm = waarde
    if waarde_alarm < 0:
        waarde_alarm = 0
    tijd = rtc.get_datetime().now() + timedelta(seconds=waarde_alarm)
    log.info("Het alarm zal afgaan om {0}".format( str(tijd)[0:19]))
    return waarde_alarm


class RotaryEncoder:
    def __init__(self, waarde=0):
        self.waarde=waarde

    def position(self):
        GPIO.setup(PIN_ROTENC_SW,GPIO.IN, pull_up_down=GPIO.PUD_UP)
        GPIO.setup(min_knop,GPIO.IN, pull_up_down=GPIO.PUD_UP)
        GPIO.setup(plus_knop,GPIO.IN, pull_up_down=GPIO.PUD_UP)
        GPIO.setup(led, GPIO.OUT)

        GPIO.add_event_detect(plus_knop, GPIO.FALLING, callback=waarde_plus)
        GPIO.add_event_detect(min_knop, GPIO.FALLING, callback = waarde_min)


    def on_press(self):
        GPIO.add_event_detect(PIN_ROTENC_SW, GPIO.RISING, callback=(rtc.set_timer))


rtc = DS3121(datetime.now())
sr = FourSevenSegment()
dis = set_value(sr)


def klok_tonen(ch=None):
    global aantal_drukken
    global waarde
    global teller
    minuten =0
    minuten_eenheden =0
    seconden = 0
    seconden_eenheden =0
    if aantal_drukken == 2:
        minuten = ((rtc.get_datetime().minute) // 10)
        minuten_eenheden = (rtc.get_datetime().minute) % 10
        seconden = (rtc.get_datetime().second // 10)
        seconden_eenheden = rtc.get_datetime().second % 10
    while True:
        teller = 0
        if gekozen_modus == 1:
            if aantal_drukken == 2:
                values = [minuten, minuten_eenheden, seconden +(waarde // 10) , seconden_eenheden + (waarde % 10)]
            else:
                values = [((rtc.get_datetime().minute) // 10), (rtc.get_datetime().minute) % 10,
                          (rtc.get_datetime().second) // 10, (rtc.get_datetime().second) % 10]
        elif gekozen_modus == 2:
            values = [((rtc.get_datetime().hour) // 10), (rtc.get_datetime().hour) % 10,
                      (rtc.get_datetime().minute) // 10, (rtc.get_datetime().minute) % 10]
        elif gekozen_modus == 3:
            values = [((rtc.get_datetime().month) // 10), (rtc.get_datetime().month) % 10,
                      (rtc.get_datetime().day) // 10, (rtc.get_datetime().day) % 10]
        for digit in PIN_DISPLAY_CATHODES:
            lights()
            if teller == 1:
                dis.show_digit_2(values[teller])
            else:
                dis.show_digit(values[teller])
            GPIO.output(digit, 1)
            time.sleep(1 / 1000)
            # felheid van de laatste digit gelijk maken aan de rest
            # time.sleep(1 / 400)
            teller += 1
        alarm_controleren()


def bcd2int(value):
        mask = 0x0f
        return (value >> 4) * 10 + (value & mask)


def int_to_bcd( value):
        result = ((value // 10) << 4) | (value % 10)
        return result


def alarm_controleren(ch=None):
    global alarm_ingesteld
    global waarde
    if alarm_ingesteld == True and (i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_STATUS)) != 0:
        log.info("Het alarm gaat op dit moment af")
        log.info(rtc.get_datetime())
        GPIO.output(led, 0)
        GPIO.output(led,1)
        time.sleep(1)
        GPIO.output(led,0)
        time.sleep(1)
        GPIO.output(led, 1)
        time.sleep(1)
        GPIO.output(led, 0)
        i2c.write_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_STATUS, (0))
        waarde = 0
        alarm_ingesteld = False


def main():
    try:
        log.info("Druk op de rotary encoder op het alarm in te stellen")
        # log.info(bcdt(i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_STATUS)))
        RotaryEncoder().position()
        RotaryEncoder().on_press()
        # log.info((i2c.read_byte_data(DS1307_DEFAULT_ADDRESS, DS3231_REG_STATUS)))
        GPIO.output(led, 0)
        trd = Thread(target=klok_tonen())
        trd.start()
    except KeyboardInterrupt:
        pass
    finally:
        GPIO.setwarnings(False)
        GPIO.cleanup()


if __name__ == '__main__':
    main()
