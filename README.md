# Quize-Learning-Platform
from random import shuffle

from django import template
from django.contrib.auth.decorators import login_required
from django.core.paginator import Paginator
from django.shortcuts import render, redirect, reverse

from quiz.models import Quiz, Question, QuizResult, UserAnswer, Answer, Course

register = template.Library()


@login_required(login_url='/user/auth/')
def take_quiz(request, *args, **kwargs):
   quiz = Quiz.objects.get(id=kwargs['pk'])
   paginator = Paginator(quiz.questions.all(), 10)
   page = request.GET.get('page', 1)
   paged_items = paginator.get_page(page)
   questions = list(quiz.questions.all())
   shuffle(questions)
   context = {
       "quiz": quiz,
       "questions": questions,
       "paged_items": paged_items,
       "objs": questions
   }
   return render(request, 'quiz/app-take-quiz.jinja2', context)


@login_required(login_url='/user/auth/')
def send_quiz(request, *args, **kwargs):
   data = dict(request.POST)
   data.pop('csrfmiddlewaretoken')
   answers = []
   score = 0
   result = QuizResult.objects.create(user=request.user,
                                      quiz_id=kwargs['pk'],
                                      score=score
                                      )

   for key, value in data.items():
       question = Question.objects.get(id=key)

       for answer in value:
           try:
               choice = question.choice.get(id=answer)
               if choice.is_correct:
                   score += 1
           except Answer.DoesNotExist:
               choice = Answer.objects.get(id=answer)

           answers.append(UserAnswer(
               question_id=question.id,
               answer_id=choice.id,
               quiz_result=result
           ))
   quiz = Quiz.objects.get(id=kwargs['pk'])

   count_of_questions = quiz.questions.count()
   score_percent = score * 100 / count_of_questions
   if score_percent >= quiz.level:
       result.finished = True
   result.score = score_percent
   result.save()
   UserAnswer.objects.bulk_create(answers)
   return redirect(reverse('course', args=[quiz.category.id]))


@login_required(login_url='/user/auth/')
def cources(request):
   quizes = Course.objects.all()
   return render(request, 'cources/app-student-courses.jinja2', {"quizes": quizes})

@login_required(login_url='/user/auth/')
def start_quiz(request, *args, **kwargs):
   course = Course.objects.get(id=kwargs['pk'])
   quizes = course.quizes.all().order_by('level')
   context = {
       'quizes': quizes,
       'course': course
   }
   return render(request, 'cources/app-take-course.jinja2', context)

@login_required(login_url='/user/auth/')
def get_course(request, *args, **kwargs):
   category = Course.objects.get(id=kwargs['pk'])
   quizes = category.quizes.all().order_by('level')
   total_score = 0
   count_quiz = 0
   for quiz in quizes:
       if quiz.quizresult.order_by('score').count() > 0:
           score = getattr(quiz.quizresult.order_by('score').last(), 'score', 0)
           total_score += score
           count_quiz += 1

   context = {
       'quizes': quizes,
       'category': category,
       'count_quiz': count_quiz,
   }
   return render(request, 'cources/app-student-course.jinja2', context)




from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.db import transaction
from django.db.models import Sum, F, Count, Avg, Max, Min
from django.shortcuts import render, redirect, reverse

from quiz.models import Course, QuizResult, Quiz
from user.forms import CustomAuthenticationForm, CustomRegistrationForm
from user.models import DocFilesUser, User


@login_required(login_url='auth')
def main(request, *args, **kwargs):
   courses = Course.objects.values("id", "name", quiz_count=Count('quizes')).filter(
       quizes__quizresult__finished=True).annotate(
       sum_result=Sum("quizes__quizresult__score") / F('quiz_count'))[:3]
   course_count = Course.objects.count()
   quizes = QuizResult.objects.filter(finished=True, user=request.user)
   test_result = quizes.aggregate(amount=Sum('score', default=0))['amount'] / quizes.count()
   sertificates = DocFilesUser.objects.filter(user=request.user)
   diagram = {
       "age": User.objects.aggregate(avg_age=Avg('age', default=0), max_age=Max('age', default=0),
                                     min_age=Min('age', default=0)),
       "gender_m": User.objects.filter(gender=1).count(),
       "gender_w": User.objects.filter(gender=2).count(),
       "score": QuizResult.objects.aggregate(max_score=Max('score', default=0),
                                             avg_score=Avg('score', default=0))
   }
   circle = Quiz.objects.all()
   context = {
       'courses': courses,
       'course_count': course_count,
       'test_result': int(test_result),
       'sertificates': sertificates,
       'diagram': diagram

   }
   # return render(request, 'base/base.jinja2')
   return render(request, 'user/dashboard.jinja2', context)


def register_view(request):
   with transaction.atomic():
       if request.method == 'POST':
           form = CustomRegistrationForm(data=request.POST)
           if form.is_valid():
               email = form.cleaned_data.get('email')
               password = form.cleaned_data.pop('password')
               user = User.objects.create(**form.cleaned_data)
               user.set_password(password)
               login(request, user)
               return redirect(reverse('dashboard'))
       else:
           form = CustomRegistrationForm()
       return render(request, 'user/sign-up.jinja2', {'form': form})


def login_view(request):
   if request.method == 'POST':
       form = CustomAuthenticationForm(data=request.POST)
       if form.is_valid():
           email = form.cleaned_data.get('email')
           password = form.cleaned_data.get('password')
           user = authenticate(request, email=email, password=password)
           if user is not None:
               login(request, user)
               return redirect(reverse('dashboard'))
   else:
       form = CustomAuthenticationForm()
   return render(request, 'user/auth_user.jinja2', {'form': form})


def profile_view(request):
   return render(request, 'user/app-student-profile.jinja2', {})
import django_summernote.fields
from django.db import models
from django_ckeditor_5.fields import CKEditor5Field

from user.models import User


class Course(models.Model):
   name = models.CharField(verbose_name='Категория курса', max_length=255)
   description = models.TextField(verbose_name='Описание категории', null=True, blank=True)
   text = CKEditor5Field('Text', config_name='extends', null=True, blank=True)
   body = models.TextField(null=True, blank=True)

   def __str__(self):
       return self.name

   class Meta:
       verbose_name = 'Курс'
       verbose_name_plural = 'Курсы'


class Quiz(models.Model):
   title = models.CharField(max_length=200, verbose_name='Тест')
   description = models.TextField(verbose_name='Описание')
   drag_drop = models.BooleanField(default=False, verbose_name='Перестаскивыемые вопросы')
   photo = models.FileField(upload_to='cource-photo', null=True, blank=True, verbose_name='Фото описание курса')
   file = models.FileField(upload_to='content/', null=True, blank=True, verbose_name='Тема для изучения')
   level = models.PositiveIntegerField(default=1, verbose_name='Уровень процента')
   category = models.ForeignKey(Course, on_delete=models.CASCADE, related_name='quizes', verbose_name='Категория',
                                null=True, blank=True)

   def get_finished_results(self):
       return self.quizresult.filter(finished=True)

   def __str__(self):
       return self.title

   class Meta:
       verbose_name = 'Тема курса'
       verbose_name_plural = 'Темы курса'


class Question(models.Model):
   quiz = models.ForeignKey(Quiz, on_delete=models.CASCADE, related_name='questions')
   description = models.TextField(blank=True, null=True, verbose_name='Детальное описание')
   question_text = models.CharField(verbose_name='Текст вопроса', max_length=200, null=True, blank=True)
   file = models.FileField(verbose_name="Файл", upload_to='question/', null=True, blank=True)

   def __str__(self):
       return self.question_text if self.question_text != "" else self.id

   class Meta:
       verbose_name = 'Вопрос'
       verbose_name_plural = 'Вопросы'

   def get_answers(self):
       return self.choice.all().order_by('?')


class Answer(models.Model):
   question = models.ForeignKey(Question, on_delete=models.CASCADE, related_name='choice')
   text = models.CharField(max_length=200, verbose_name='Текст ответа', null=True, blank=True)
   file = models.FileField(verbose_name="Файл", upload_to='answer/', null=True, blank=True)
   is_correct = models.BooleanField(default=False, verbose_name='Правильный ответ')

   def __str__(self):
       if self.is_correct:
           return f"{self.question} - {self.text}"
       return f"{self.text}"

   class Meta:
       verbose_name = 'Ответ'
       verbose_name_plural = 'Ответы'


class QuizResult(models.Model):
   user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="quizresult")
   quiz = models.ForeignKey(Quiz, on_delete=models.CASCADE, related_name="quizresult")
   score = models.IntegerField()
   finished = models.BooleanField(default=False, verbose_name='Успешно пройден')
   date = models.DateTimeField(auto_now_add=True)

   def __str__(self):
       return f"{self.user.email} - {self.score}"

   class Meta:
       verbose_name = 'Результаты теста'
       verbose_name_plural = 'Результаты теста'


class UserAnswer(models.Model):
   quiz_result = models.ForeignKey(QuizResult, on_delete=models.CASCADE, related_name='user_answer')
   question = models.ForeignKey(Question, on_delete=models.CASCADE, related_name='user_answer')
   answer = models.ForeignKey(Answer, on_delete=models.CASCADE, related_name='user_answer')

   def __str__(self):
       return f"{self.quiz_result}"

   class Meta:
       verbose_name = 'Выбранные ответы теста'
       verbose_name_plural = 'Выбранные ответы теста'


from email.policy import default

from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):

   type_gender = (
       (1, 'Мальчик'),
       (2, 'Девочка')
   )

   username = models.CharField(max_length=50, blank=True, null=True, unique=True)
   email = models.EmailField(verbose_name='email address', unique=True)
   gender = models.IntegerField(choices=type_gender, default=1)
   phone = models.CharField(max_length=11)
   photo = models.ImageField(upload_to="users/", null=True, blank=True)
   age = models.IntegerField(default=18)
   ent_result = models.IntegerField(verbose_name='Результат ент', null=True, blank=True)

   USERNAME_FIELD = 'email'
   REQUIRED_FIELDS = ['username', 'first_name', 'last_name']

   def __str__(self):
       return "{}".format(self.email)

   class Meta:
       verbose_name = 'Студент'
       verbose_name_plural = 'Студенты'


class DocFilesUser(models.Model):
   file = models.FileField(verbose_name='Файлы')
   user = models.ForeignKey(User, verbose_name='Пользователь', on_delete=models.CASCADE)


from django.urls import path, include

from user.views import main, login_view, profile_view, register_view

urlpatterns = [
   path('', include([
       path('auth/', login_view, name='auth'),
       path('register/', register_view, name='register'),
       path('profile', profile_view, name='profile'),
       path('logout/', login_view, name='logout'),
       path('', main, name='dashboard'),
   ]))

]









from django.urls import path

from quiz.views import take_quiz, send_quiz, cources, start_quiz, get_course

urlpatterns = [
   path('courses/', cources, name='courses'),
   path('course/<int:pk>/', get_course, name='course'),
   path('course/<int:pk>/start/', start_quiz, name='start-course'),
   path('take-quiz/<int:pk>/start/', take_quiz, name='start-quiz'),
   path('take-quiz/<int:pk>/start/save/', send_quiz, name='send-quiz'),
]



#!/user/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
   """Run administrative tasks."""
   os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core1.settings')
   try:
       from django.core.management import execute_from_command_line
   except ImportError as exc:
       raise ImportError(
           "Couldn't import Django. Are you sure it's installed and "
           "available on your PYTHONPATH environment variable? Did you "
           "forget to activate a virtual environment?"
       ) from exc
   execute_from_command_line(sys.argv)


if __name__ == '__main__':
   main()


"""
Django settings for core1 project.

Generated by 'django-admin startproject' using Django 4.2.

For more information on this file, see
https://docs.djangoproject.com/en/4.2/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/4.2/ref/settings/
"""

from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/4.2/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'django-insecure-c1z+2nzblp#0*ax*j+804nmlaxfnt&$2f*@&@4u_rmk-+1xk*n'

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = []

# Application definition


INSTALLED_APPS = [
   'django.contrib.admin',
   'django.contrib.auth',
   'django.contrib.contenttypes',
   'django.contrib.sessions',
   'django.contrib.messages',
   'django.contrib.staticfiles',
   'quiz.apps.QuizConfig',
   'user',
   "corsheaders",
   'django_summernote',
   'django_ckeditor_5'
]

MIDDLEWARE = [
   'django.middleware.locale.LocaleMiddleware',
   'django.middleware.security.SecurityMiddleware',
   'django.contrib.sessions.middleware.SessionMiddleware',
   'corsheaders.middleware.CorsMiddleware',
   'django.middleware.common.CommonMiddleware',
   'django.middleware.csrf.CsrfViewMiddleware',
   'django.contrib.auth.middleware.AuthenticationMiddleware',
   'django.contrib.messages.middleware.MessageMiddleware',
   'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'core1.urls'

TEMPLATES = [
   {
       'BACKEND': 'django.template.backends.django.DjangoTemplates',
       'DIRS': [BASE_DIR / 'templates']
       ,
       'APP_DIRS': True,
       'OPTIONS': {
           'context_processors': [
               'django.template.context_processors.debug',
               'django.template.context_processors.request',
               'django.contrib.auth.context_processors.auth',
               'django.contrib.messages.context_processors.messages',
           ],
       },
   },
]
AUTH_USER_MODEL = 'user.User'
WSGI_APPLICATION = 'core1.wsgi.application'

# Database
# https://docs.djangoproject.com/en/4.2/ref/settings/#databases

DATABASES = {
   'default': {
       'ENGINE': 'django.db.backends.sqlite3',
       'NAME': BASE_DIR / 'db.sqlite3',
   }
}

# Password validation
# https://docs.djangoproject.com/en/4.2/ref/settings/#auth-password-validators

AUTH_PASSWORD_VALIDATORS = [
   {
       'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
   },
   {
       'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
   },
   {
       'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
   },
   {
       'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
   },
]

# Internationalization
# https://docs.djangoproject.com/en/4.2/topics/i18n/

LANGUAGES = [
   ('ru', ('Russian')),
   ('en', ('English')),
]

LANGUAGE_CODE = 'ru'

TIME_ZONE = 'Asia/Almaty'

USE_I18N = True

USE_TZ = True

# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/4.2/howto/static-files/

STATIC_URL = 'static/'
STATICFILES_DIRS = (
   BASE_DIR, "static"
)

STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media/'

# Default primary key field type
# https://docs.djangoproject.com/en/4.2/ref/settings/#default-auto-field

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

CORS_ORIGIN_ALLOW_ALL = True
LOGIN_URL = 'user/auth/'
LOGIN_REDIRECT_URL = 'user/dashboard/'

customColorPalette = [
   {"color": "hsl(4, 90%, 58%)", "label": "Red"},
   {"color": "hsl(340, 82%, 52%)", "label": "Pink"},
   {"color": "hsl(291, 64%, 42%)", "label": "Purple"},
   {"color": "hsl(262, 52%, 47%)", "label": "Deep Purple"},
   {"color": "hsl(231, 48%, 48%)", "label": "Indigo"},
   {"color": "hsl(207, 90%, 54%)", "label": "Blue"},
]

CKEDITOR_5_CONFIGS = {
   "default": {
       'language': 'ru',
       "toolbar": [
           "heading",
           "|",
           "bold",
           "italic",
           "link",
           "bulletedList",
           "numberedList",
           "blockQuote",
           "imageUpload"
       ],
   },
   "comment": {
       "language": {"ui": "en", "content": "en"},
       "toolbar": [
           "heading",
           "|",
           "bold",
           "italic",
           "link",
           "bulletedList",
           "numberedList",
           "blockQuote",
       ],
   },
   "extends": {
       "language": "en",
       "blockToolbar": [
           "paragraph",
           "heading1",
           "heading2",
           "heading3",
           "|",
           "bulletedList",
           "numberedList",
           "|",
           "blockQuote",
       ],
       "toolbar": [
           "heading",
           "codeBlock",
           "|",
           "outdent",
           "indent",
           "|",
           "bold",
           "italic",
           "link",
           "underline",
           "strikethrough",
           "code",
           "subscript",
           "superscript",
           "highlight",
           "|",
           "bulletedList",
           "numberedList",
           "todoList",
           "|",
           "blockQuote",
           "insertImage",
           "|",
           "fontSize",
           "fontFamily",
           "fontColor",
           "fontBackgroundColor",
           "mediaEmbed",
           "removeFormat",
           "insertTable",
           "sourceEditing",
       ],
       "image": {
           "toolbar": [
               "imageTextAlternative",
               "|",
               "imageStyle:alignLeft",
               "imageStyle:alignRight",
               "imageStyle:alignCenter",
               "imageStyle:side",
               "|",
               "toggleImageCaption",
               "|"
           ],
           "styles": [
               "full",
               "side",
               "alignLeft",
               "alignRight",
               "alignCenter",
           ],
       },
       "table": {
           "contentToolbar": [
               "tableColumn",
               "tableRow",
               "mergeTableCells",
               "tableProperties",
               "tableCellProperties",
           ],
           "tableProperties": {
               "borderColors": customColorPalette,
               "backgroundColors": customColorPalette,
           },
           "tableCellProperties": {
               "borderColors": customColorPalette,
               "backgroundColors": customColorPalette,
           },
       },
       "heading": {
           "options": [
               {
                   "model": "paragraph",
                   "title": "Paragraph",
                   "class": "ck-heading_paragraph",
               },
               {
                   "model": "heading1",
                   "view": "h1",
                   "title": "Heading 1",
                   "class": "ck-heading_heading1",
               },
               {
                   "model": "heading2",
                   "view": "h2",
                   "title": "Heading 2",
                   "class": "ck-heading_heading2",
               },
               {
                   "model": "heading3",
                   "view": "h3",
                   "title": "Heading 3",
                   "class": "ck-heading_heading3",
               },
           ]
       },
       "list": {
           "properties": {
               "styles": True,
               "startIndex": True,
               "reversed": True,
           }
       },
       "htmlSupport": {
           "allow": [
               {"name": "/.*/", "attributes": True, "classes": True, "styles": True}
           ]
       },
   },
}


SUMMERNOTE_THEME = 'bs4'  # Show summernote with Bootstrap4


SUMMERNOTE_CONFIG = {
   # Using SummernoteWidget - iframe mode, default
   'iframe': True,

}

