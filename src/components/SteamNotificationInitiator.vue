<template>
  
</template>

<script>
import firebase from 'firebase/app'
import 'firebase/database'
import 'firebase/firestore'
import 'firebase/functions'
import db from '@/database.js'
import DatabaseHelpersMixin from '@/mixins/DatabaseHelpersMixin.js'
import { mapState } from 'vuex'

export default {
  mixins: [
    DatabaseHelpersMixin
  ],
  data () {
    return {
      steamNotificationTimeout: null
    }
  },
  computed: {
    ...mapState([
      'user',
      'mitClass'
    ]),
    // FUTURE TODO: make these getters() instead
    userIsArmyMember () {
      const { user, mitClass } = this 
      if (!user.classesVolunteered) return false 
      return user.classesVolunteered.includes(mitClass.id)
    },
    userIsTA () {
      const { user, mitClass } = this
      if (!user.classesTAing) return false
      return user.classesTAing.includes(mitClass.id)
    },
  },
  async created () {
    if (!this.userIsArmyMember) {
      return
    }

    const { user, mitClass } = this
    // check if 60 minutes has passed since last steam notification
    const { lastSteamNotificationTimestamp } = user
    if (lastSteamNotificationTimestamp) {
      const SIXTY_MINUTES_IN_MILLISECONDS = 60 * 60 * 1000
      if (Date.now() - lastSteamNotificationTimestamp.toDate().getTime() < SIXTY_MINUTES_IN_MILLISECONDS) {
        return
      }
    }

    // notify other members of "Army of help" but do a 5 seconds delay to ensure it's not just a random visit
    this.steamNotificationTimeout = setTimeout(async () => {
      db.doc(`users/${user.uid}`).update({
        lastSteamNotificationTimestamp: firebase.firestore.FieldValue.serverTimestamp()
      })
      // all these only apply to volunteers
      // so if they want to halt participation, they will not get notifications
      // but when they resume, they get their settings back
      const armyMembers = await this.$_getCollection(
        db.collection('users').where('classesVolunteered', 'array-contains', this.mitClass.id)
      )
      let peopleToNotify
      let emailTitle
      let emailContent
      const { class_id, section_id, room_id } = this.$route.params
      if (user.kind === 'staff' || this.userIsTA) {
        emailTitle = `${user.firstName + ' ' + user.lastName } just logged on to ${this.mitClass.name}`
        emailContent = `
          Join link:
          <a href="${window.location.origin}/class/${class_id}/section/${section_id}/room/${room_id}">
            ${window.location.origin}/class/${class_id}/section/${section_id}/room/${room_id}
          </a>
          <p>Unsubscribe by adjusting "Army of Helpers" settings</p>
          <p>
            Get push notifications (iOS only) by pressing "From: eltonlin@mit.edu" near the top, then press "Add to VIP". 
          </p>
        `
        peopleToNotify = armyMembers.filter(member => { 
          if (!member.getNotifiedWhenInstructorsComeOnline) return false 
          return member.getNotifiedWhenInstructorsComeOnline.includes(mitClass.id)
        })
      } else {
        emailTitle = `${user.firstName} just logged on to ${this.mitClass.name}`

        // original redirect link: "https://explain.web.app/class/${class_id}/section/${section_id}/room/${room_id}"
        emailContent = `
          Join link:
          <a href="${window.location.origin}/class/${class_id}/section/${section_id}/room/${room_id}">
            ${window.location.origin}/class/${class_id}/section/${section_id}/room/${room_id}
          </a>
          <p>Unsubscribe by adjusting "Army of Helpers" settings</p>
          <p>
            Get push notifications (iOS only) by pressing "From: eltonlin@mit.edu" near the top, then press "Add to VIP". 
          </p>
        `
        peopleToNotify = armyMembers.filter(member => { 
          if (!member.getNotifiedWhenClassmatesComeOnline) return false 
          return member.getNotifiedWhenClassmatesComeOnline.includes(mitClass.id)
        })
      }
      
      for (const person of peopleToNotify) {
        const sendEmailToPerson = firebase.functions().httpsCallable('sendEmailToPerson');
        console.log('send email to =', person.email)
        sendEmailToPerson({ 
          emailOfPerson: person.email, 
          title: emailTitle, 
          contentHTML: emailContent
        })
      }
    }, 20000)
  },
  destroyed () {
    if (this.steamNotificationTimeout) clearTimeout(this.steamNotificationTimeout) 
  }
}
</script>