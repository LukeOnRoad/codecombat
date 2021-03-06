<template lang="pug">
  flat-layout
    .container.m-t-3
      p
        a(href="/students", data-i18n="courses.back_courses")
      div.m-t-2
        h2.text-center
          | {{ $t('courses.challenges') }}
        h1(v-if="classroom").text-center
          | {{ classroom.name }}
        select(v-model="selectedCourse")
          option(v-for="chunk in levelsByCourse", :value="chunk.course._id") {{ $dbt(chunk.course, 'name') }}
        div.assessments-list.m-t-3(v-for="chunk in levelsByCourse" v-if="chunk.course._id === selectedCourse")
          .row
            .col-xs-5
              span.table-header
                | {{ $dbt(chunk.course, 'name') }}
            .status-column.col-xs-5
              span.table-header
                | {{ $t('courses.status') }}
          student-assessment-row(
            v-for="level in chunk.assessmentLevels",
            :assessment="level.assessment",
            :assessmentPlacement="level.assessmentPlacement",
            :primaryConcept="level.primaryConcept",
            :name="$dbt(level, 'name')",
            :complete="!!(sessionMap[level.original] && sessionMap[level.original].state.complete)",
            :started="!!sessionMap[level.original]",
            :playUrl="playLevelUrlMap[level.original]",
            :goals="level.goals",
            :goalStates="sessionMap[level.original] ? sessionMap[level.original].state.goalStates : []"
          )

</template>

<script lang="coffee">

FlatLayout = require 'core/components/FlatLayout'
api = require 'core/api'
User = require 'models/User'
Level = require 'models/Level'
LevelSession = require 'models/LevelSession'
utils = require 'core/utils'
StudentAssessmentRow = require('./StudentAssessmentRow').default
  
module.exports = Vue.extend
  name: 'student-assessments-component'
  components:
    'flat-layout': FlatLayout,
    'student-assessment-row': StudentAssessmentRow
  props:
    classroomID:
      type: String
      default: -> null
  data: ->
    courseInstances: []
    levelSessions: []
    classroom: null
    levels: null
    levelsByCourse: null
    sessionMap: {}
    playLevelUrlMap: {}
    levelUnlockedMap: {}
    inCourses: {}
    courses: []
    selectedCourse: ''
  computed:
    backToClassroomUrl: -> "/teachers/classes/#{@classroom?._id}"
  created: ->
    # TODO: Only fetch the ones for this classroom
    Promise.all([
      # TODO: Only load the levels we actually need
      Promise.all([
        api.users.getCourseInstances({ userID: me.id }).then((@courseInstances) =>),
        api.courses.getAll().then((@courses) =>),
        api.classrooms.get({ @classroomID }, { data: {memberID: me.id}, cache: false }).then((@classroom) =>)
      ]).then(=>
        @allLevels = _.flatten(_.map(@classroom.courses, (course) => course.levels))
        @levels = _.flatten(_.map(@classroom.courses, (course) => _.filter(course.levels, 'assessment')))
        @inCourses = {}
        for courseInstance in @courseInstances
          if courseInstance.classroomID is @classroomID and me.id in courseInstance.members
            @inCourses[courseInstance.courseID] = true
        @levelsByCourse = _.map(@classroom.courses, (course) => {
          course: _.find(@courses, ({_id: course._id})),
          assessmentLevels: _.filter(course.levels, 'assessment')
        }).filter((chunk) => chunk.assessmentLevels.length and @inCourses[chunk.course._id])
        @selectedCourse = document.location.hash.replace('#','') or _.first(@levelsByCourse)?.course._id
        
        @courses = @classroom.courses
        return Promise.all(_.map(@levels, (level) =>
          api.levels.getByOriginal(level.original, {
            data: { project: 'slug,name,original,primaryConcepts,i18n,goals' }
          }).then (data) =>
            levelToUpdate = _.find(@levels, {original: data.original})
            Vue.set(levelToUpdate, 'primaryConcept', _.first(data.primaryConcepts))
            Vue.set(levelToUpdate, 'i18n', data.i18n)
            Vue.set(levelToUpdate, 'goals', data.goals)
        ))
      )
      api.users.getLevelSessions({ userID: me.id }).then((@levelSessions) =>)
    ]).then =>
      @sessionMap = @createSessionMap()
      @playLevelUrlMap = @createPlayLevelUrlMap()
      # These two maps are for determining if a challenge is unlocked yet
      @previousLevelMap = @createPreviousLevelMap()
      @levelUnlockedMap = @createLevelUnlockedMap()
  methods:
    createSessionMap: ->
      # Map level original to levelSession
      _.reduce(@levelSessions, (map, session) =>
        map[session.level.original] = session
        return map
      , {})
    createPlayLevelUrlMap: ->
      # Map level original to URL to play that level as the student
      _.reduce(@levels, (map, level) =>
        course = _.find(@courses, (c) =>
          Boolean(_.find(c.levels, (l) => l.original is level.original))
        )
        courseInstance = _.find(@courseInstances, (ci) => ci.courseID is course._id and ci.classroomID is @classroomID)
        if _.all([level.slug, courseInstance?._id, course?._id])
          map[level.original] = "/play/level/#{level.slug}?course-instance=#{courseInstance?._id}&course=#{course?._id}"
        return map
      , {})
    createPreviousLevelMap: ->
      # Map assessment original to the level original of the level that unlocks the assessment
      map = {}
      for level, index in @allLevels
        assessmentIndex = utils.findNextAssessmentForLevel(@allLevels, index)
        if assessmentIndex isnt false
          assessmentOriginal = @allLevels[assessmentIndex].original
          map[assessmentOriginal] ?= level.original
      return map
    createLevelUnlockedMap: ->
      # Map assessment original to whether it has been unlocked yet, using session for the previous level
      map = {}
      for level, index in @levels
        map[level.original] = @sessionMap[@previousLevelMap[level.original]]?.state.complete or false
      return map  
  watch: {
    selectedCourse: (newValue) ->
      document.location.hash = newValue
  }
</script>

<style lang="sass">
#student-assessments-view .style-flat
  .table-header
    font-weight: bold

  font-size: 16px
  line-height: 27px
</style>
