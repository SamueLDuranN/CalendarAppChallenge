from datetime import datetime, date, time
from typing import ClassVar, List, Dict, Optional

from app.services.util import generate_unique_id, date_lower_than_today_error, event_not_found_error, \
    reminder_not_found_error, slot_not_available_error


# TODO: Implement Reminder class here
class Reminder:
    EMAIL: ClassVar[str] = 'email'
    SYSTEM: ClassVar[str] = 'system'

    def __init__(self, date_time: datetime, type: str = EMAIL):
        self.date_time = date_time
        self.type = type

    def __str__(self):
        return f"Reminder on {self.date_time} of type {self.type}"


# TODO: Implement Event class here
class Event:
    def __init__(self, title: str, description: str, date_: date, start_at: time, end_at: time):
        self.title = title
        self.description = description
        self.date_ = date_
        self.start_at = start_at
        self.end_at = end_at
        self.id = generate_unique_id()
        self.reminders: List[Reminder] = []

    def add_reminder(self, date_time: datetime, type_: str):
        reminder = Reminder(date_time=date_time, type=type_)
        self.reminders.append(reminder)

    def delete_reminder(self, reminder_index: int):
        if 0 <= reminder_index < len(self.reminders):
            self.reminders.pop(reminder_index)
        else:
            reminder_not_found_error()

    def __str__(self):
        return (f"ID: {self.id}\n"
                f"Event title: {self.title}\n"
                f"Description: {self.description}\n"
                f"Time: {self.start_at} - {self.end_at}")


# TODO: Implement Day class here
class Day:
    def __init__(self, date_: date):
        self.date_ = date_
        self.slots: Dict[time, Optional[str]] = {}
        self._init_slots()

    def _init_slots(self):
        # Inicializa los slots en bloques de 15 minutos
        for hour in range(24):
            for minute in range(0, 60, 15):
                self.slots[time(hour, minute)] = None

    def add_event(self, event_id: str, start_at: time, end_at: time):
        # Agrega un evento a los slots dentro del rango de start_at y end_at (sin incluir end_at)
        for slot in self.slots:
            if start_at <= slot < end_at:
                if self.slots[slot] is not None:
                    slot_not_available_error()  # El slot ya está ocupado
                self.slots[slot] = event_id

    def delete_event(self, event_id: str):
        deleted = False
        for slot, saved_id in self.slots.items():
            if saved_id == event_id:
                self.slots[slot] = None
                deleted = True
        if not deleted:
            event_not_found_error()

    def update_event(self, event_id: str, start_at: time, end_at: time):
        # Elimina los slots antiguos del evento
        for slot in self.slots:
            if self.slots[slot] == event_id:
                self.slots[slot] = None

        # Agrega los nuevos slots
        for slot in self.slots:
            if start_at <= slot < end_at:
                if self.slots[slot] is not None:
                    slot_not_available_error()
                self.slots[slot] = event_id


# TODO: Implement Calendar class here
class Calendar:
    def __init__(self):
        self.days: Dict[date, Day] = {}
        self.events: Dict[str, Event] = {}

    def add_event(self, title: str, description: str, date_: date, start_at: time, end_at: time) -> str:
        # Verifica que la fecha no sea anterior a la actual
        if date_ < datetime.now().date():
            date_lower_than_today_error()

        # Si el día no existe, lo crea
        if date_ not in self.days:
            self.days[date_] = Day(date_)

        # Crea el evento
        event = Event(title=title, description=description, date_=date_, start_at=start_at, end_at=end_at)
        self.days[date_].add_event(event.id, start_at, end_at)
        self.events[event.id] = event

        return event.id

    def update_event(self, event_id: str, title: str, description: str, date_: date, start_at: time, end_at: time):
        # Encuentra el evento
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()

        # Verifica si la fecha cambió
        is_new_date = event.date_ != date_

        # Si cambió de fecha, elimina el evento de la fecha anterior
        if is_new_date:
            self.delete_event(event_id)
            event = Event(title=title, description=description, date_=date_, start_at=start_at, end_at=end_at)
            event.id = event_id
            self.events[event_id] = event
            if date_ not in self.days:
                self.days[date_] = Day(date_)
            self.days[date_].add_event(event_id, start_at, end_at)
        else:
            # Si la fecha no cambia, actualiza el evento
            event.title = title
            event.description = description
            event.start_at = start_at
            event.end_at = end_at

        # Actualiza los slots del día
        for day in self.days.values():
            if not is_new_date and event_id in day.slots.values():
                day.delete_event(event_id)
                day.update_event(event_id, start_at, end_at)

    def delete_event(self, event_id: str):
        # Elimina el evento del diccionario
        if event_id not in self.events:
            event_not_found_error()

        self.events.pop(event_id)

        # Elimina el evento de los slots
        for day in self.days.values():
            if event_id in day.slots.values():
                day.delete_event(event_id)
                break

    def find_events(self, start_at: date, end_at: date) -> Dict[date, List[Event]]:
        events: Dict[date, List[Event]] = {}
        for event in self.events.values():
            if start_at <= event.date_ <= end_at:
                if event.date_ not in events:
                    events[event.date_] = []
                events[event.date_].append(event)
        return events

    def add_reminder(self, event_id: str, date_time: datetime, type_: str):
        # Encuentra el evento
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()

        event.add_reminder(date_time=date_time, type_=type_)

    def delete_reminder(self, event_id: str, reminder_index: int):
        # Encuentra el evento
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()

        event.delete_reminder(reminder_index)

    def list_reminders(self, event_id: str) -> List[Reminder]:
        # Encuentra el evento
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()

        return event.reminders

    def find_available_slots(self, date_: date) -> List[time]:
        # Encuentra los slots disponibles en un día
        if date_ not in self.days:
            return [time(hour, minute) for hour in range(24) for minute in range(0, 60, 15)]

        available_slots = [slot for slot, event_id in self.days[date_].slots.items() if event_id is None]
        return available_slots
