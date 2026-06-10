import { useEffect, useMemo, useState } from 'react';
import LoginScreen from '../screens/auth/LoginScreen';
import DashboardScreen from '../screens/dashboard/DashboardScreen';
import CourseDetailScreen from '../screens/courses/CourseDetailScreen';
import CourseListScreen from '../screens/courses/CourseListScreen';
import SafetyTipsScreen from '../screens/hazards/SafetyTipsScreen';
import ProgressDashboardScreen from '../screens/progress/ProgressDashboardScreen';
import CertificateHistoryScreen from '../screens/certificates/CertificateHistoryScreen';
import {
  isFirebaseConfigured,
  missingFirebaseConfigKeys,
} from '../services/firebase';
import {
  loginFirebaseAccount,
  registerFirebaseAccount,
  resetFirebasePassword,
  signOutFirebaseAccount,
  watchFirebaseUser,
} from '../services/accountService';
import {
  fetchCourses,
  fetchCourseProgress,
  fetchSafetyTips,
  mergeCourseProgress,
  saveCourseProgress,
} from '../services/trainingService';
import {
  printCertificate as printCertificateDocument,
  recordCertificate,
  recordQuizAttempt,
} from '../services/certificateService';

const screens = {
  Login: LoginScreen,
  Dashboard: DashboardScreen,
  Courses: CourseListScreen,
  CourseDetail: CourseDetailScreen,
  SafetyTips: SafetyTipsScreen,
  Progress: ProgressDashboardScreen,
  Certificates: CertificateHistoryScreen,
};

export default function AppNavigator() {
  const [stack, setStack] = useState([{ name: 'Login', params: {} }]);
  const [currentUser, setCurrentUser] = useState(null);
  const [courses, setCourses] = useState([]);
  const [safetyTips, setSafetyTips] = useState([]);
  const [trainingLoaded, setTrainingLoaded] = useState(false);
  const [trainingError, setTrainingError] = useState('');
  const [isPersistingTraining, setIsPersistingTraining] = useState(false);
  const current = stack[stack.length - 1];
  const Screen = screens[current.name] || DashboardScreen;

  async function loadTraining(uid) {
    setTrainingLoaded(false);
    setTrainingError('');
    try {
      if (!uid) {
        setCourses([]);
        setSafetyTips([]);
        setTrainingLoaded(true);
        return;
      }

      if (!isFirebaseConfigured) {
        setCourses([]);
        setSafetyTips([]);
        setTrainingError(`Firebase is missing required config: ${missingFirebaseConfigKeys.join(', ')}`);
        setTrainingLoaded(true);
        return;
      }

      const [allCourses, progressDoc, allSafetyTips] = await Promise.all([
        fetchCourses(),
        fetchCourseProgress(uid),
        fetchSafetyTips(),
      ]);

      if (allCourses.length === 0) {
        throw new Error('No courses found in Firestore. Seed the courses collection, then reload.');
      }

      if (allSafetyTips.length === 0) {
        throw new Error('No safety tips found in Firestore. Seed the safetyTips collection, then reload.');
      }

      const merged = mergeCourseProgress(allCourses, progressDoc);
      setCourses(merged);
      setSafetyTips(allSafetyTips);
    } catch (error) {
      setCourses([]);
      setSafetyTips([]);
      setTrainingError(error.message || 'Could not load training data from Firebase.');
      console.error('Could not load training data:', error.message || error);
    } finally {
      setTrainingLoaded(true);
    }
  }

  useEffect(() => {
    return watchFirebaseUser((user) => {
      setCurrentUser(user);
      if (user) {
        setStack([{ name: 'Dashboard', params: {} }]);
      } else {
        setCourses([]);
        setSafetyTips([]);
        setTrainingLoaded(false);
        setTrainingError('');
      }
    });
  }, []);

  useEffect(() => {
    if (currentUser?.uid && isFirebaseConfigured) {
      loadTraining(currentUser.uid);
    }
  }, [currentUser?.uid]);

  const navigation = useMemo(
    () => ({
      navigate: (name, params = {}) => {
        setStack((entries) => [...entries, { name, params }]);
      },
      goBack: () => {
        setStack((entries) => (entries.length > 1 ? entries.slice(0, -1) : entries));
      },
      resetToDashboard: (user) => {
        if (user) {
          setCurrentUser(user);
        }
        setStack([{ name: 'Dashboard', params: {} }]);
      },
      resetToLogin: async () => {
        await signOutFirebaseAccount();
        setCurrentUser(null);
        setCourses([]);
        setSafetyTips([]);
        setTrainingLoaded(false);
        setTrainingError('');
        setStack([{ name: 'Login', params: {} }]);
      },
    }),
    []
  );

  const auth = useMemo(
    () => ({
      currentUser,
      isFirebaseConfigured,
      firebaseConfigError: isFirebaseConfigured
        ? ''
        : `Missing Firebase config: ${missingFirebaseConfigKeys.join(', ')}`,
      registerAccount: async (account) => {
        if (!isFirebaseConfigured) {
          return { ok: false, message: `Firebase is not configured: ${missingFirebaseConfigKeys.join(', ')}` };
        }

        const result = await registerFirebaseAccount(account);
        if (result.ok) {
          setCurrentUser(result.user);
          setStack([{ name: 'Dashboard', params: {} }]);
          await loadTraining(result.user.uid);
        }
        return result;
      },
      loginAccount: async ({ identifier, password }) => {
        if (!isFirebaseConfigured) {
          return { ok: false, message: `Firebase is not configured: ${missingFirebaseConfigKeys.join(', ')}` };
        }

        const result = await loginFirebaseAccount({ identifier, password });
        if (result.ok) {
          setCurrentUser(result.user);
          setStack([{ name: 'Dashboard', params: {} }]);
          await loadTraining(result.user.uid);
        }
        return result;
      },
      resetPassword: async (email) => {
        if (!isFirebaseConfigured) {
          return { ok: false, message: `Firebase is not configured: ${missingFirebaseConfigKeys.join(', ')}` };
        }

        return resetFirebasePassword(email);
      },
    }),
    [currentUser]
  );

  const training = useMemo(() => {
    const getProgressMap = (items) => {
      return items.reduce((result, course) => {
        const completedTitles = course.lessons.filter((lesson) => lesson.completed).map((lesson) => lesson.title);
        const quizScores = course.lessons.reduce((scores, lesson) => {
          if (lesson.assessment && lesson.completed && typeof lesson.savedScore === 'number') {
            scores[lesson.title] = lesson.savedScore;
          }
          return scores;
        }, {});
        if (completedTitles.length > 0) {
          result[course.id] = {
            completedLessonTitles: completedTitles,
            completedLessons: completedTitles.length,
            progress: course.progress,
            locked: course.progress >= 100,
            quizScores,
          };
        }
        return result;
      }, {});
    };

    const completeLesson = async (courseId, lessonTitle, lessonObject, score = 100) => {
      const course = courses.find((item) => item.id === courseId);
      if (!course || lessonObject?.completed) {
        return { ok: true, completedCourse: false };
      }

      if (course.progress >= 100) {
        return {
          ok: false,
          completedCourse: false,
          message: 'This course is already complete and locked. You can review the memo or print the certificate, but it cannot be redone.',
        };
      }

      if (!currentUser?.uid || !isFirebaseConfigured) {
        setTrainingError('Firebase sign-in is required before progress can be saved.');
        return { ok: false, message: 'Firebase sign-in is required before progress can be saved.' };
      }

      const updatedCourses = courses.map((item) => {
        if (item.id !== courseId) {
          return item;
        }

        const lessons = item.lessons.map((lesson) =>
          lesson.title === lessonTitle
            ? {
                ...lesson,
                completed: true,
                ...(lessonObject?.assessment ? { savedScore: score } : {}),
              }
            : lesson
        );
        const completedLessons = lessons.filter((lesson) => lesson.completed).length;
        const progress = Math.round((completedLessons / lessons.length) * 100);

        return {
          ...item,
          lessons,
          completedLessons,
          progress,
        };
      });

      const previousCourses = courses;
      setCourses(updatedCourses);
      setIsPersistingTraining(true);
      setTrainingError('');

      try {
        const progressData = getProgressMap(updatedCourses);
        await saveCourseProgress(currentUser.uid, progressData);

        if (lessonObject?.assessment) {
          await recordQuizAttempt(currentUser.uid, courseId, score);
        }

        const updatedCourse = updatedCourses.find((item) => item.id === courseId);
        const completedCourse = course.progress < 100 && updatedCourse?.progress === 100;
        if (course.progress < 100 && updatedCourse?.progress === 100) {
          await recordCertificate(currentUser.uid, updatedCourse, currentUser);
        }
        return { ok: true, completedCourse };
      } catch (error) {
        setCourses(previousCourses);
        setTrainingError(error.message || 'Could not save training progress to Firebase.');
        console.error('Could not persist training progress:', error.message || error);
        return { ok: false, message: error.message || 'Could not save training progress to Firebase.' };
      } finally {
        setIsPersistingTraining(false);
      }
    };

    const printCertificate = async (course) => {
      if (!currentUser) {
        return;
      }
      try {
        await printCertificateDocument(course, currentUser);
      } catch (error) {
        console.error('Could not print certificate:', error.message || error);
      }
    };

    const overallProgress = courses.length
      ? Math.round(courses.reduce((sum, course) => sum + (course.progress || 0), 0) / courses.length)
      : 0;
    const examScores = courses.flatMap((course) =>
      (course.lessons || [])
        .filter((lesson) => lesson.assessment && typeof lesson.savedScore === 'number')
        .map((lesson) => lesson.savedScore)
    );
    const examAverage = examScores.length
      ? Math.round(examScores.reduce((sum, score) => sum + score, 0) / examScores.length)
      : overallProgress;
    const safetyLevel = overallProgress >= 75 ? 'Advanced' : overallProgress >= 35 ? 'Intermediate' : 'Beginner';

    return {
      courses,
      safetyTips,
      completeLesson,
      printCertificate,
      overallProgress,
      safetyLevel,
      quizAverage: examAverage,
      trainingLoaded,
      trainingError,
      isPersistingTraining,
      reloadTraining: () => loadTraining(currentUser?.uid),
    };
  }, [courses, safetyTips, currentUser, trainingLoaded, trainingError, isPersistingTraining]);

  return <Screen navigation={navigation} route={{ params: current.params }} auth={auth} training={training} />;
}

