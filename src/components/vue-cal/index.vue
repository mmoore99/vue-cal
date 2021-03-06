<template lang="pug">
  .vuecal__flex.vuecal(column :class="cssClasses" :lang="locale")
    vuecal-header(
      :vuecal-props="$props"
      :view-props="{ views, view, hasSplits }"
      :months="months"
      :week-days="weekDays"
      :week-days-short="weekDaysShort")
      slot(slot="arrowPrev" name="arrowPrev")
        i.angle
      slot(slot="arrowNext" name="arrowNext")
        i.angle
      slot(slot="title" name="title" :title="viewTitle" :view="view") {{ viewTitle }}

    .vuecal__flex.vuecal__body(v-if="!hideBody" grow)
      transition(:name="`slide-fade--${transitionDirection}`" :appear="transitions")
        .vuecal__flex(style="min-width: 100%" :key="transitions ? view.id : false" column)
          .vuecal__flex.vuecal__all-day(v-if="showAllDayEvents && hasTimeColumn")
            span(style="width: 3em")
              span {{ texts.allDay }}
            .vuecal__flex.vuecal__cells(:class="`${view.id}-view`" grow :wrap="!hasSplits || view.id !== 'week'" :column="hasSplits")
              vuecal-cell(
                :class="cell.class"
                v-for="(cell, i) in viewCells"
                :key="i"
                :date="cell.date"
                :formatted-date="cell.formattedDate"
                :all-day-events="true"
                :today="cell.today"
                :splits="hasSplits && splitDays || []"
                @click.native="selectCell(cell)"
                @dblclick.native="dblClickToNavigate && switchToNarrowerView()")
                div(slot="cell-content")
                  slot(name="cell-content" :cell="cell" :view="view" :switchToNarrowerView="switchToNarrowerView")
                div(slot="event-renderer" slot-scope="{ event, view }" :view="view" :event="event")
                  slot(name="event-renderer" :view="view" :event="event")
                    .vuecal__event-title.vuecal__event-title--edit(
                      v-if="editableEvents && event.title"
                      contenteditable
                      @blur="onEventTitleBlur($event, event)"
                      v-html="event.title")
                    .vuecal__event-title(v-else-if="event.title" v-html="event.title")
                    .vuecal__event-content(
                      v-if="event.content && showAllDayEvents !== 'short' && !isShortMonthView"
                      v-html="event.content")
                slot(slot="no-event" name="no-event")
          .vuecal__bg(:class="{ vuecal__flex: !hasTimeColumn }" column)
            .vuecal__flex(row grow)
              .vuecal__time-column(v-if="hasTimeColumn")
                .vuecal__time-cell(v-for="(cell, i) in timeCells" :key="i" :style="`height: ${timeCellHeight}px`")
                  slot(name="time-cell" :hours="cell.hours" :minutes="cell.minutes")
                    span.line {{ cell.label }}

              .vuecal__flex.vuecal__cells(:class="`${view.id}-view`" grow :wrap="!hasSplits || view.id !== 'week'" :column="hasSplits")
                //- Only for splitDays.
                weekdays-headings(
                  v-if="hasSplits && view.id === 'week'"
                  :transitions="{ active: transitions, direction: transitionDirection }"
                  :view="view"
                  :min-cell-width="minCellWidth"
                  :week-days="weekDays"
                  :week-days-short="weekDaysShort")

                .vuecal__flex(grow :wrap="!hasSplits || view.id !== 'week'")
                  vuecal-cell(
                    :class="cell.class"
                    v-for="(cell, i) in viewCells"
                    :key="i"
                    :date="cell.date"
                    :formatted-date="cell.formattedDate"
                    :today="cell.today"
                    :content="cell.content"
                    :splits="hasSplits && splitDays || []"
                    @click.native="selectCell(cell)"
                    @dblclick.native="dblClickToNavigate && switchToNarrowerView()")
                    div(slot="cell-content" slot-scope="{ events }")
                      slot(name="cell-content" :cell="cell" :view="view" :goNarrower="switchToNarrowerView" :events="events")
                        .vuecal__cell-date(v-if="cell.content" v-html="cell.content")
                        .vuecal__cell-events-count(v-if="view.id === 'month' && !eventsOnMonthView && events.length") {{ events.length }}
                        .vuecal__no-event(v-if="!events.length && ['week', 'day'].includes(view.id)")
                          slot(name="no-event") {{ texts.noEvent }}
                    div(slot="event-renderer" slot-scope="{ event, view }" :view="view" :event="event")
                      slot(name="event-renderer" :view="view" :event="event")
                        .vuecal__event-title.vuecal__event-title--edit(
                          v-if="editableEvents && event.title"
                          contenteditable
                          @blur="onEventTitleBlur($event, event)"
                          v-html="event.title")
                        .vuecal__event-title(v-else-if="event.title" v-html="event.title")
                        .vuecal__event-time(v-if="event.startTimeMinutes && !(view === 'month' && event.allDay && showAllDayEvents === 'short') && !isShortMonthView")
                          | {{ event.startTimeMinutes | formatTime(timeFormat || ($props['12Hour'] ? 'h:mm{am}' : 'HH:mm')) }}
                          span(v-if="event.endTimeMinutes") &nbsp;- {{ event.endTimeMinutes | formatTime(timeFormat || ($props['12Hour'] ? 'h:mm{am}' : 'HH:mm')) }}
                          small.days-to-end(v-if="event.multipleDays.daysCount") &nbsp;+{{ event.multipleDays.daysCount - 1 }}{{ (texts.day[0] || '').toLowerCase() }}
                        .vuecal__event-content(
                          v-if="event.content && !(view === 'month' && event.allDay && showAllDayEvents === 'short') && !isShortMonthView"
                          v-html="event.content")
                    slot(slot="no-event" name="no-event") {{ texts.noEvent }}
</template>

<script>
import { now, isDateToday, getPreviousFirstDayOfWeek, formatDate, formatTime } from './date-utils'
import { onResizeEvent } from './event-utils'
import Header from './header'
import WeekdaysHeadings from './weekdays-headings'
import Cell from './cell'
import './styles.scss'

export default {
  name: 'vue-cal',
  components: { 'vuecal-cell': Cell, 'vuecal-header': Header, WeekdaysHeadings },
  props: {
    locale: {
      type: String,
      default: 'en'
    },
    hideViewSelector: {
      type: Boolean,
      default: false
    },
    hideTitleBar: {
      type: Boolean,
      default: false
    },
    hideBody: {
      type: Boolean,
      default: false
    },
    hideWeekends: {
      type: Boolean,
      default: false
    },
    disableViews: {
      type: Array,
      default: () => []
    },
    defaultView: {
      type: String,
      default: 'week'
    },
    showAllDayEvents: {
      type: [Boolean, String],
      default: false
    },
    selectedDate: {
      type: [String, Date],
      default: ''
    },
    startWeekOnSunday: {
      type: Boolean,
      default: false
    },
    small: {
      type: Boolean,
      default: false
    },
    xsmall: {
      type: Boolean,
      default: false
    },
    clickToNavigate: {
      type: Boolean,
      default: false
    },
    dblClickToNavigate: {
      type: Boolean,
      default: true
    },
    time: {
      type: Boolean,
      default: true
    },
    timeFrom: {
      type: Number,
      default: 0 // In minutes.
    },
    timeTo: {
      type: Number,
      default: 24 * 60 // In minutes.
    },
    timeStep: {
      type: Number,
      default: 60 // In minutes.
    },
    timeCellHeight: {
      type: Number,
      default: 40 // In pixels.
    },
    '12Hour': {
      type: Boolean,
      default: false
    },
    timeFormat: {
      type: String,
      default: ''
    },
    minCellWidth: {
      type: Number,
      default: 0
    },
    splitDays: {
      type: Array,
      default: () => []
    },
    events: {
      type: Array,
      default: () => []
    },
    editableEvents: {
      type: Boolean,
      default: false
    },
    noEventOverlaps: {
      type: Boolean,
      default: false
    },
    eventsOnMonthView: {
      type: [Boolean, String],
      default: false
    },
    onEventClick: {
      type: Function,
      default: () => {}
    },
    onEventDblclick: {
      type: Function,
      default: () => {}
    },
    transitions: {
      type: Boolean,
      default: true
    }
  },
  data: function () {
    return {
      texts: this.locale === 'en' ? require('./i18n/en.json') : {
        weekDays: Array(7).fill(''),
        months: Array(12).fill(''),
        years: '',
        year: '',
        month: '',
        week: '',
        day: '',
        today: '',
        noEvent: '',
        allDay: '',
        deleteEvent: '',
        createEvent: '',
        dateFormat: 'DDDD mmmm d, yyyy'
      },
      ready: false,
      now,
      view: {
        id: '',
        title: '',
        startDate: null,
        selectedDate: null
      },
      eventIdIncrement: 1,
      domEvents: {
        resizeAnEvent: {
          eid: null, // Only one at a time.
          startDate: null,
          start: null,
          originalHeight: 0,
          newHeight: 0,
          split: null
        },
        dragAnEvent: {
          eid: null // Only one at a time.
        },
        focusAnEvent: {
          eid: null // Only one at a time.
        },
        clickHoldAnEvent: {
          eid: null, // Only one at a time.
          timeout: 1200,
          timeoutId: null
        },
        dblTapACell: {
          taps: 0,
          timeout: 500
        }
      },
      mutableEvents: {}, // An indexed array of mutable events updated each time given events array changes.
      transitionDirection: 'right'
    }
  },

  methods: {
    loadLocale (locale) {
      // import(/* webpackInclude: /\.json$/, webpackChunkName: "[request]" */ `./i18n/${locale}`)
      //   .then(response => (this.texts = response.default))
      this.texts = require(`./i18n/${locale}.json`)
    },

    switchToNarrowerView () {
      this.transitionDirection = 'right'

      let views = Object.keys(this.views)
      views = views.slice(views.indexOf(this.view.id) + 1)
      const view = views.find(v => this.views[v].enabled)

      if (view) this.switchView(view)
    },

    switchView (view, date = null, fromViewSelector = false) {
      if (this.transitions && fromViewSelector) {
        const views = Object.keys(this.views)
        this.transitionDirection = views.indexOf(this.view.id) > views.indexOf(view) ? 'left' : 'right'
      }

      this.view.events = []
      this.view.id = view
      this.view.firstCellDate = null // For month view, if filling cells before 1st of month.
      this.view.lastCellDate = null // For month view, if filling cells after current month.
      let dateTmp, endTime, formattedDate, dayEvents

      if (!date) {
        date = this.view.selectedDate || this.view.startDate
        if (view === 'week') date = getPreviousFirstDayOfWeek(date, this.startWeekOnSunday)
      }

      switch (view) {
        case 'years':
          // Always fill first cell with a multiple of 25 years, E.g. year 2000, or 2025.
          this.view.startDate = new Date(Math.floor(date.getFullYear() / 25) * 25 || 2000, 0, 1)
          this.view.endDate = new Date(this.view.startDate.getFullYear() + 26, 0, 0)
          break
        case 'year':
          this.view.startDate = new Date(date.getFullYear(), 0, 1)
          this.view.endDate = new Date(date.getFullYear() + 1, 0, 0)
          break
        case 'month':
          this.view.startDate = new Date(date.getFullYear(), date.getMonth(), 1)
          this.view.endDate = new Date(date.getFullYear(), date.getMonth() + 1, 0)
          dateTmp = new Date(this.view.startDate)
          endTime = this.view.endDate.getTime()
          while (dateTmp.getTime() <= endTime) {
            dateTmp = dateTmp.addDays(1)
            formattedDate = formatDate(dateTmp, 'yyyy-mm-dd', this.texts)

            // If the first day of the month is not a FirstDayOfWeek (Monday or Sunday), prepend missing days to the days array.
            let startDate = new Date(this.view.startDate)
            if (startDate.getDay() !== (this.startWeekOnSunday ? 0 : 1)) {
              startDate = getPreviousFirstDayOfWeek(startDate, this.startWeekOnSunday)
            }

            // Used in viewCells computed array & returned in emitted events.
            this.view.firstCellDate = startDate
            this.view.lastCellDate = startDate.addDays(41)

            // Save the events of each day of month + out of scope ones into view object.
            for (let i = 0; i < 42; i++) { // 42 cells (6 rows x 7 days).
              formattedDate = formatDate(startDate.addDays(i), 'yyyy-mm-dd', this.texts)
              dayEvents = this.mutableEvents[formattedDate] || []
              if (dayEvents.length) this.view.events.push(...dayEvents.map(e => this.cleanupEvent(e)))
            }
          }
          break
        case 'week':
          this.view.startDate = this.hideWeekends && this.startWeekOnSunday ? date.addDays(1) : date
          this.view.endDate = date.addDays(7)
          dateTmp = new Date(date)
          for (let i = 0; i < 7; i++) {
            formattedDate = formatDate(dateTmp.addDays(i), 'yyyy-mm-dd', this.texts)
            dayEvents = this.mutableEvents[formattedDate] || []
            if (dayEvents.length) this.view.events.push(...dayEvents.map(e => this.cleanupEvent(e)))
          }
          break
        case 'day':
          this.view.startDate = date
          this.view.endDate = date
          dayEvents = this.mutableEvents[formatDate(date, 'yyyy-mm-dd', this.texts)] || []
          if (dayEvents.length) this.view.events = dayEvents.map(e => this.cleanupEvent(e))
          break
      }

      if (this.ready) {
        const params = {
          view,
          startDate: this.view.startDate,
          endDate: this.view.endDate,
          ...(this.view.id === 'month' ? { firstCellDate: this.view.firstCellDate, lastCellDate: this.view.lastCellDate } : {}),
          events: this.view.events,
          ...(this.view.id === 'week' ? { week: this.view.startDate.getWeek() } : {})
        }
        this.$emit('view-change', params)
      }
    },

    findAncestor (el, Class) {
      while ((el = el.parentElement) && !el.classList.contains(Class)) {}
      return el
    },

    isDOMElementAnEvent (el) {
      return el.classList.contains('vuecal__event') || this.findAncestor(el, 'vuecal__event')
    },

    selectCell (cell) {
      this.$emit('day-click', cell.date)
      if (this.view.selectedDate.toString() !== cell.date.toString()) {
        this.view.selectedDate = cell.date
        this.$emit('day-focus', cell.date)
      }

      // Switch to narrower view.
      if (this.clickToNavigate) this.switchToNarrowerView()

      // Handle double click manually for touch devices.
      else if (this.dblClickToNavigate && 'ontouchstart' in window) {
        this.domEvents.dblTapACell.taps++

        setTimeout(() => (this.domEvents.dblTapACell.taps = 0), this.domEvents.dblTapACell.timeout)

        if (this.domEvents.dblTapACell.taps >= 2) {
          this.domEvents.dblTapACell.taps = 0
          this.switchToNarrowerView()
        }
      }
    },

    // Event resizing is started in cell component (onMouseDown) but place onMouseMove & onMouseUp
    // handlers in the single parent for performance.
    onMouseMove (e) {
      let { resizeAnEvent } = this.domEvents
      if (resizeAnEvent.eid === null) return

      e.preventDefault()
      const y = 'ontouchstart' in window ? e.touches[0].clientY : e.clientY
      resizeAnEvent.newHeight = resizeAnEvent.originalHeight + (y - resizeAnEvent.start)

      let cellEvents = this.mutableEvents[resizeAnEvent.startDate]
      if (this.hasSplits && this.splitDays) {
        cellEvents = cellEvents.filter(e => e.split === resizeAnEvent.split)
      }
      onResizeEvent(cellEvents, this)
    },

    onMouseUp (e) {
      let { focusAnEvent, resizeAnEvent, clickHoldAnEvent } = this.domEvents

      // On event resize end, emit event if duration has changed.
      if (resizeAnEvent.eid && resizeAnEvent.newHeight !== resizeAnEvent.originalHeight) {
        let event = this.mutableEvents[resizeAnEvent.startDate].find(item => item.eid === resizeAnEvent.eid)
        if (event) {
          this.emitWithEvent('event-change', event)
          this.emitWithEvent('event-duration-change', event)
        }
      }

      // If not mouse up on an event, unfocus any event except if just dragged.
      if (!this.isDOMElementAnEvent(e.target) && !resizeAnEvent.eid) {
        focusAnEvent.eid = null // Cancel event focus.
        clickHoldAnEvent.eid = null // Hide delete button.
      }

      // Prevent showing delete button if click and hold was not long enough.
      // Click & hold timeout happens in onMouseDown() in cell component.
      if (clickHoldAnEvent.timeoutId && !clickHoldAnEvent.eid) {
        clearTimeout(clickHoldAnEvent.timeoutId)
        clickHoldAnEvent.timeoutId = null
      }

      // Any mouse up must cancel event resizing.
      resizeAnEvent.eid = null
      resizeAnEvent.start = null
      resizeAnEvent.originalHeight = null
      resizeAnEvent.newHeight = null
    },

    onEventTitleBlur (e, event) {
      // If no change cancel action.
      if (event.title === e.target.innerHTML) return

      event.title = e.target.innerHTML

      if (event.linked.daysCount) {
        event.linked.forEach(e => {
          let dayToModify = this.mutableEvents[e.date]
          dayToModify.find(e2 => e2.eid === e.eid).title = event.title
        })
      }

      this.emitWithEvent('event-change', event)
      this.emitWithEvent('event-title-change', event)
    },

    // Object of arrays of events indexed by dates.
    updateMutableEvents () {
      // eslint-disable-next-line
      this.mutableEvents = {}

      // Group events into dates.
      this.events.map(event => {
        let [startDate, startTime = ''] = event.start.split(' ')
        const [hoursStart, minutesStart] = startTime.split(':')
        const startTimeMinutes = parseInt(hoursStart) * 60 + parseInt(minutesStart)

        let [endDate, endTime = ''] = event.end.split(' ')
        const [hoursEnd, minutesEnd] = endTime.split(':')
        const endTimeMinutes = parseInt(hoursEnd) * 60 + parseInt(minutesEnd)
        const multipleDays = startDate !== endDate

        // Keep the event ids scoped to this calendar instance.
        // eslint-disable-next-line
        let eid = `${this._uid}_${this.eventIdIncrement++}`

        event = Object.assign({
          eid,
          startDate,
          startTime,
          startTimeMinutes,
          endDate,
          endTime,
          endTimeMinutes,
          title: '',
          content: '',
          height: 0,
          top: 0,
          overlapped: {},
          overlapping: {},
          simultaneous: {},
          linked: [], // Linked events.
          multipleDays: {},
          allDay: false,
          classes: (event.class || '').split(' ')
        }, event)

        // Make array reactive for future events creations & deletions.
        if (!(event.startDate in this.mutableEvents)) this.$set(this.mutableEvents, event.startDate, [])
        // eslint-disable-next-line
        this.mutableEvents[event.startDate].push(event)

        if (multipleDays) {
          // Create an array of linked events to attach to each event piece (piece = 1 day),
          // So that deletion and modification reflects on all the pieces.
          let eventPieces = []

          const oneDayInMs = 24 * 60 * 60 * 1000
          const [y1, m1, d1] = startDate.split('-')
          const [y2, m2, d2] = endDate.split('-')
          startDate = new Date(y1, parseInt(m1) - 1, d1)
          endDate = new Date(y2, parseInt(m2) - 1, d2)
          const datesDiff = Math.round(Math.abs((startDate.getTime() - endDate.getTime()) / oneDayInMs))
          const startDateFormatted = formatDate(startDate, 'yyyy-mm-dd', this.texts)

          // Update First day event.
          event.multipleDays = {
            start: true,
            startDate: startDateFormatted,
            startTime,
            startTimeMinutes,
            endDate: startDateFormatted,
            endTime: '24:00',
            endTimeMinutes: 24 * 60,
            daysCount: datesDiff + 1
          }

          // Generate event pieces ids to link them all together
          // and update the first event linked events array with all ids of pieces.
          for (let i = 1; i <= datesDiff; i++) {
            const date = formatDate(new Date(startDate).addDays(i), 'yyyy-mm-dd', this.texts)
            eventPieces.push({
              eid: `${this._uid}_${this.eventIdIncrement++}`,
              date
            })
          }
          event.linked = eventPieces

          // Create 1 event per day and link them all.
          for (let i = 1; i <= datesDiff; i++) {
            const date = eventPieces[i - 1].date
            const linked = [
              { eid: event.eid, date: event.startDate },
              ...eventPieces.slice(0).filter(e => e.eid !== eventPieces[i - 1].eid)
            ]

            // Make array reactive for future events creations & deletions.
            if (!(date in this.mutableEvents)) this.$set(this.mutableEvents, date, [])

            this.mutableEvents[date].push({
              ...event,
              eid: eventPieces[i - 1].eid,
              overlapped: {},
              overlapping: {},
              simultaneous: {},
              linked,
              // All the dates in the multipleDays object property are related
              // to the current event piece (only 1 day) not the whole event.
              multipleDays: {
                start: false,
                middle: i < datesDiff,
                end: i === datesDiff,
                startDate: date,
                startTime: '00:00',
                startTimeMinutes: 0,
                endDate: date,
                endTime: i === datesDiff ? endTime : '24:00',
                endTimeMinutes: i === datesDiff ? endTimeMinutes : 24 * 60,
                daysCount: datesDiff + 1
              }
            })
          }
        }

        return event
      })
    },

    // Prepare the event to return it with an emitted event.
    cleanupEvent (event) {
      event = { ...event }

      // Delete vue-cal specific props instead of returning a set of props so user
      // can place whatever they want inside an event and see it returned.
      const discardProps = ['height', 'top', 'overlapped', 'overlapping', 'simultaneous', 'classes', 'split']
      for (let prop in event) if (discardProps.indexOf(prop) > -1) delete event[prop]
      if (!event.multipleDays.daysCount) delete event.multipleDays

      // Return date objects for easy manipulation.
      const [date1, time1] = event.start.split(' ')
      const [y1, m1, d1] = (date1 && date1.split('-')) || [0, 0, 0]
      const [h1, min1] = (time1 && time1.split(':')) || [0, 0]
      event.startDate = new Date(y1, parseInt(m1) - 1, d1, h1, min1)

      const [date2, time2] = event.end.split(' ')
      const [y2, m2, d2] = (date2 && date2.split('-')) || [0, 0, 0]
      const [h2, min2] = (time2 && time2.split(':')) || [0, 0]
      event.endDate = new Date(y2, parseInt(m2) - 1, d2, h2, min2)

      return event
    },

    emitWithEvent (eventName, event) {
      this.$emit(eventName, this.cleanupEvent(event))
    },

    updateSelectedDate (date) {
      if (date && typeof date === 'string') {
        let [, y, m, d, h = 0, min = 0] = date.match(/(\d{4})-(\d{2})-(\d{2})(?: (\d{2}):(\d{2}))?/)
        date = new Date(y, parseInt(m) - 1, d, h, min)
      }
      if (date && (typeof date === 'string' || date instanceof Date)) {
        if (this.view.selectedDate) this.transitionDirection = this.view.selectedDate.getTime() > date.getTime() ? 'left' : 'right'
        this.view.selectedDate = date
        this.switchView(this.view.id)
      }
    }
  },

  created () {
    if (this.locale !== 'en') this.loadLocale(this.locale)

    // Init the array of events, then keep listening for changes in watcher.
    this.updateMutableEvents(this.events)

    this.view.id = this.defaultView
    if (this.selectedDate) this.updateSelectedDate(this.selectedDate)
    else {
      this.view.selectedDate = this.now
      this.switchView(this.defaultView)
    }
  },

  mounted () {
    if (this.editableEvents) {
      const hasTouch = 'ontouchstart' in window
      window.addEventListener(hasTouch ? 'touchmove' : 'mousemove', this.onMouseMove, { passive: false })
      window.addEventListener(hasTouch ? 'touchend' : 'mouseup', this.onMouseUp)
    }

    const params = {
      view: this.view.id,
      startDate: this.view.startDate,
      endDate: this.view.endDate,
      ...(this.view.id === 'month' ? { firstCellDate: this.view.firstCellDate, lastCellDate: this.view.lastCellDate } : {}),
      events: this.view.events,
      ...(this.view.id === 'week' ? { week: this.view.startDate.getWeek() } : {})
    }

    this.$emit('ready', params)
    this.ready = true
  },

  beforeDestroy () {
    const hasTouch = 'ontouchstart' in window
    window.removeEventListener(hasTouch ? 'touchmove' : 'mousemove', this.onMouseMove, { passive: false })
    window.removeEventListener(hasTouch ? 'touchend' : 'mouseup', this.onMouseUp)
  },

  computed: {
    views () {
      return {
        years: { label: this.texts.years, enabled: this.disableViews.indexOf('years') === -1 },
        year: { label: this.texts.year, enabled: this.disableViews.indexOf('year') === -1 },
        month: { label: this.texts.month, enabled: this.disableViews.indexOf('month') === -1 },
        week: { label: this.texts.week, enabled: this.disableViews.indexOf('week') === -1 },
        day: { label: this.texts.day, enabled: this.disableViews.indexOf('day') === -1 }
      }
    },
    hasTimeColumn () {
      return this.time && ['week', 'day'].indexOf(this.view.id) > -1
    },
    isShortMonthView () {
      return this.view.id === 'month' && this.eventsOnMonthView === 'short'
    },
    // For week & day views.
    timeCells () {
      let timeCells = []
      for (let i = this.timeFrom, max = this.timeTo; i < max; i += this.timeStep) {
        timeCells.push({
          hours: Math.floor(i / 60),
          minutes: i % 60,
          label: formatTime(i, this.timeFormat || (this['12Hour'] ? 'h:mm{am}' : 'HH:mm')),
          value: i
        })
      }

      return timeCells
    },
    // Whether the current view has days splits.
    hasSplits () {
      return !!this.splitDays.length && ['week', 'day'].indexOf(this.view.id) > -1
    },
    weekDays () {
      let { weekDays } = this.texts
      // Do not modify original for next instances.
      weekDays = weekDays.slice(0)

      if (this.startWeekOnSunday) weekDays.unshift(weekDays.pop())

      if (this.hideWeekends) {
        weekDays = this.startWeekOnSunday ? weekDays.slice(1, 6) : weekDays.slice(0, 5)
      }

      return weekDays.map(day => ({ label: day }))
    },
    weekDaysShort () {
      let { weekDaysShort = null } = this.texts

      if (weekDaysShort) {
        // Do not modify original for next instances.
        weekDaysShort = weekDaysShort.slice(0)

        if (this.startWeekOnSunday) weekDaysShort.unshift(weekDaysShort.pop())

        if (this.hideWeekends) {
          weekDaysShort = this.startWeekOnSunday ? weekDaysShort.slice(1, 6) : weekDaysShort.slice(0, 5)
        }
      }

      return weekDaysShort && weekDaysShort.map(day => ({ label: day }))
    },
    months () {
      return this.texts.months.map(month => ({ label: month }))
    },
    viewTitle () {
      let title = ''
      const date = this.view.startDate
      const year = date.getFullYear()
      const month = date.getMonth()

      switch (this.view.id) {
        case 'years':
          title = this.texts.years
          break
        case 'year':
          title = year
          break
        case 'month':
          title = `${this.months[month].label} ${year}`
          break
        case 'week':
          const lastDayOfWeek = date.addDays(6)
          let formattedMonthYear = formatDate(date, this.xsmall ? 'mmm yyyy' : 'mmmm yyyy', this.texts)

          // If week is not ending in the same month it started in.
          if (lastDayOfWeek.getMonth() !== date.getMonth()) {
            let [m1, y1] = formattedMonthYear.split(' ')
            let [m2, y2] = formatDate(lastDayOfWeek, this.xsmall ? 'mmm yyyy' : 'mmmm yyyy', this.texts).split(' ')
            formattedMonthYear = y1 === y2 ? `${m1} - ${m2} ${y1}` : `${m1} ${y1} - ${m2} ${y2}`
          }
          title = `${this.texts.week} ${date.getWeek()} (${formattedMonthYear})`
          break
        case 'day':
          title = formatDate(date, this.texts.dateFormat, this.texts)
          break
      }

      return title
    },
    viewCells () {
      let cells = []
      let fromYear = null
      let todayFound = false

      switch (this.view.id) {
        case 'years':
          fromYear = this.view.startDate.getFullYear()
          cells = Array.apply(null, Array(25)).map((cell, i) => {
            return {
              content: fromYear + i,
              date: new Date(fromYear + i, 0, 1),
              class: {
                current: fromYear + i === this.now.getFullYear(),
                selected: this.view.selectedDate && (fromYear + i) === this.view.selectedDate.getFullYear()
              }
            }
          })
          break
        case 'year':
          fromYear = this.view.startDate.getFullYear()
          cells = Array.apply(null, Array(12)).map((cell, i) => {
            return {
              content: this.xsmall ? this.months[i].label.substr(0, 3) : this.months[i].label,
              date: new Date(fromYear, i, 1),
              class: {
                current: i === this.now.getMonth() && fromYear === this.now.getFullYear(),
                selected: i === this.view.selectedDate.getMonth() && fromYear === this.view.selectedDate.getFullYear()
              }
            }
          })
          break
        case 'month':
          const month = this.view.startDate.getMonth()
          let selectedDateAtMidnight = new Date(this.view.selectedDate.getTime())
          selectedDateAtMidnight.setHours(0, 0, 0, 0)
          todayFound = false
          let startDate = new Date(this.view.firstCellDate)

          // Create 42 cells (6 rows x 7 days) and populate them with days.
          cells = Array.apply(null, Array(42)).map((cell, i) => {
            const cellDate = startDate.addDays(i)
            // To increase performance skip checking isToday if today already found.
            const isToday = !todayFound && cellDate && cellDate.getDate() === this.now.getDate() &&
                            cellDate.getMonth() === this.now.getMonth() &&
                            cellDate.getFullYear() === this.now.getFullYear() &&
                            !todayFound++
            const formattedDate = formatDate(cellDate, 'yyyy-mm-dd', this.texts)

            return {
              content: cellDate.getDate(),
              date: cellDate,
              formattedDate,
              today: isToday,
              class: {
                today: isToday,
                'out-of-scope': cellDate.getMonth() !== month,
                selected: this.view.selectedDate && cellDate.getTime() === selectedDateAtMidnight.getTime()
              }
            }
          })

          if (this.hideWeekends) {
            cells = cells.filter(cell => cell.date.getDay() > 0 && cell.date.getDay() < 6)
          }
          break
        case 'week':
          todayFound = false
          let firstDayOfWeek = this.view.startDate

          cells = this.weekDays.map((cell, i) => {
            const date = firstDayOfWeek.addDays(i)
            const formattedDate = formatDate(date, 'yyyy-mm-dd', this.texts)
            let isToday = !todayFound && isDateToday(date) && !todayFound++

            return {
              date,
              formattedDate,
              today: isToday,
              class: {
                today: isToday,
                selected: this.view.selectedDate && firstDayOfWeek.addDays(i).getTime() === this.view.selectedDate.getTime()
              }
            }
          })
          break
        case 'day':
          const formattedDate = formatDate(this.view.startDate, 'yyyy-mm-dd', this.texts)
          const isToday = isDateToday(this.view.startDate)

          cells = [{
            date: this.view.startDate,
            formattedDate,
            today: isToday,
            class: {
              today: isToday,
              selected: this.view.selectedDate && this.view.startDate.getTime() === this.view.selectedDate.getTime()
            }
          }]
          break
      }
      return cells
    },
    cssClasses () {
      return {
        [`vuecal--${this.view.id}-view`]: true,
        [`vuecal--${this.locale}`]: this.locale,
        'vuecal--no-time': !this.time,
        'vuecal--view-with-time': this.hasTimeColumn,
        'vuecal--time-12-hour': this['12Hour'],
        'vuecal--click-to-navigate': this.clickToNavigate,
        'vuecal--hide-weekends': this.hideWeekends,
        'vuecal--split-days': this.hasSplits,
        'vuecal--overflow-x': this.minCellWidth,
        'vuecal--small': this.small,
        'vuecal--xsmall': this.xsmall,
        'vuecal--no-event-overlaps': this.noEventOverlaps,
        'vuecal--dragging-event': this.domEvents.resizeAnEvent.start,
        'vuecal--events-on-month-view': this.eventsOnMonthView,
        'vuecal--short-events': this.view.id === 'month' && this.eventsOnMonthView === 'short'
      }
    }
  },

  filters: {
    formatTime (value, format) {
      return (value && (formatTime(value, format) || ''))
    }
  },

  watch: {
    events: {
      handler (events, oldEvents) {
        this.updateMutableEvents(events)
      },
      deep: true
    },
    locale (locale) {
      this.loadLocale(locale)
    },
    selectedDate (date) {
      this.updateSelectedDate(date)
    }
  }
}
</script>
