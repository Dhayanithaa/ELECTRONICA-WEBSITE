# ELECTRONICA-WEBSITE
It contains the py files for the website
#MAIN.py
import random
from datetime import datetime, timedelta

from fastapi import FastAPI, HTTPException, Header
from sqlalchemy.exc import IntegrityError

from database import Base, engine, SessionLocal

import models
import schemas


# DATABASE


Base.metadata.create_all(bind=engine)



# APPLICATION

app = FastAPI(
    title="ELECTRONICA Circuit Challenge API",
    version="1.0"
)




def get_settings(db):

    settings = db.query(
        models.CompetitionSettings
    ).filter(
        models.CompetitionSettings.id == 1
    ).first()

    if settings is None:

        settings = models.CompetitionSettings(
            id=1,
            active_round="round 1",
            round_duration_minutes=45
        )

        db.add(settings)
        db.commit()
        db.refresh(settings)

    return settings


def authenticate_team(db, access_token):

    if not access_token:

        raise HTTPException(
            status_code=401,
            detail="Access token required"
        )

    team = db.query(models.Team).filter(
        models.Team.access_token == access_token
    ).first()

    if team is None:

        raise HTTPException(
            status_code=401,
            detail="Invalid access token"
        )

    return team


def require_admin(db, access_token):

    team = authenticate_team(
        db,
        access_token
    )

    if team.role.lower() != "admin":

        raise HTTPException(
            status_code=403,
            detail="Admin access required"
        )

    return team


def get_or_create_attempt(
    db,
    team,
    round_name
):

    attempt = db.query(
        models.RoundAttempt
    ).filter(
        models.RoundAttempt.team_id == team.team_id,
        models.RoundAttempt.round == round_name
    ).first()

    if attempt:
        return attempt

    settings = get_settings(db)

    now = datetime.utcnow()

    attempt = models.RoundAttempt(
        team_id=team.team_id,
        round=round_name,
        started_at=now,
        deadline=now + timedelta(
            minutes=settings.round_duration_minutes
        )
    )

    db.add(attempt)
    db.commit()
    db.refresh(attempt)

    return attempt


def ensure_attempt_open(attempt):

    if attempt.final_submitted:

        raise HTTPException(
            status_code=400,
            detail="Attempt has already been submitted"
        )

    if datetime.utcnow() > attempt.deadline:

        raise HTTPException(
            status_code=400,
            detail="Time is over for this attempt"
        )


# HOME


@app.get("/")
def home():

    return {
        "message": "Electronica backend is running"
    }



# LOGIN

@app.post("/login")
def login(data: schemas.LoginRequest):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            data.access_token
        )

        settings = get_settings(db)

        return {
            "team_id": team.team_id,
            "team_name": team.team_name,
            "role": team.role,
            "status": team.status,
            "active_round": settings.active_round
        }

    finally:
        db.close()



# ADMIN - CREATE TEAM


@app.post("/admin/teams")
def create_team(
    data: schemas.TeamCreate,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        role = data.role.lower().strip()

        if role not in ["team", "admin"]:

            raise HTTPException(
                status_code=400,
                detail="role must be team or admin"
            )

        status = data.status.lower().strip()

        if status not in ["round 1", "round 2"]:

            raise HTTPException(
                status_code=400,
                detail="Invalid team status"
            )

        team = models.Team(
            team_name=data.team_name.strip(),
            access_token=data.access_token.strip(),
            role=role,
            status=status
        )

        db.add(team)

        try:
            db.commit()

        except IntegrityError:

            db.rollback()

            raise HTTPException(
                status_code=400,
                detail="Access token already exists"
            )

        db.refresh(team)

        return {
            "team_id": team.team_id,
            "team_name": team.team_name,
            "role": team.role,
            "status": team.status
        }

    finally:
        db.close()


# ADMIN - TEAM STATUS

@app.patch("/admin/teams/{team_id}/status")
def update_team_status(
    team_id: str,
    data: schemas.TeamStatusUpdate,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        status = data.status.lower().strip()

        if status not in [
            "round 1",
            "round 2"
        ]:

            raise HTTPException(
                status_code=400,
                detail="Invalid status"
            )

        team = db.query(models.Team).filter(
            models.Team.team_id == team_id
        ).first()

        if not team:

            raise HTTPException(
                status_code=404,
                detail="Team not found"
            )

        team.status = status

        db.commit()

        return {
            "message": "Team status updated",
            "team_id": team.team_id,
            "status": team.status
        }

    finally:
        db.close()


# ADMIN - COMPETITION SETTINGS


@app.get("/competition/settings")
def competition_settings():

    db = SessionLocal()

    try:

        settings = get_settings(db)

        return {
            "active_round": settings.active_round,
            "round_duration_minutes":
                settings.round_duration_minutes,
            "updated_at": settings.updated_at
        }

    finally:
        db.close()


@app.patch("/admin/competition/settings")
def change_competition_settings(
    data: schemas.RoundSwitch,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        round_name = data.active_round.lower().strip()

        if round_name not in [
            "round 1",
            "round 2",
            "locked"
        ]:

            raise HTTPException(
                status_code=400,
                detail="Invalid active round"
            )

        settings = get_settings(db)

        settings.active_round = round_name

        if data.round_duration_minutes is not None:

            settings.round_duration_minutes = (
                data.round_duration_minutes
            )

        settings.updated_at = datetime.utcnow()

        db.commit()

        return {
            "message": "Competition settings updated",
            "active_round": settings.active_round,
            "round_duration_minutes":
                settings.round_duration_minutes
        }

    finally:
        db.close()



# ADMIN - ROUND 1 QUESTIONS

@app.post("/admin/round1/questions")
def create_round1_question(
    data: schemas.Round1QuestionCreate,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        difficulty = (
            data.level_of_difficulty
            .lower()
            .strip()
        )

        if difficulty not in [
            "easy",
            "medium",
            "hard"
        ]:

            raise HTTPException(
                status_code=400,
                detail="Difficulty must be easy, medium or hard"
            )

        correct = (
            data.correct_answer_id
            .upper()
            .strip()
        )

        option_ids = {
            str(option.get("option_id", "")).upper()
            for option in data.options
        }

        if correct not in option_ids:

            raise HTTPException(
                status_code=400,
                detail="Correct answer must exist in options"
            )

        question = models.Round1Question(
            question_number=data.question_number,
            question=data.question,
            options=data.options,
            pic=data.pic,
            correct_answer_id=correct,
            level_of_difficulty=difficulty,
            marks=data.marks
        )

        db.add(question)
        db.commit()
        db.refresh(question)

        return {
            "message": "Round 1 question created",
            "id": question.id
        }

    finally:
        db.close()


# ADMIN - ROUND 2 QUESTIONS


@app.post("/admin/round2/questions")
def create_round2_question(
    data: schemas.Round2QuestionCreate,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        question = models.Round2Question(
            question_number=data.question_number,
            question=data.question,
            pic=data.pic,
            tool=data.tool,
            evaluation_criteria=data.evaluation_criteria
        )

        db.add(question)
        db.commit()
        db.refresh(question)

        return {
            "message": "Round 2 question created",
            "id": question.id
        }

    finally:
        db.close()


# ROUND 1 START / RESUME


@app.post("/round1/start")
def start_round1(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        if team.role != "team":

            raise HTTPException(
                status_code=403,
                detail="Team account required"
            )

        settings = get_settings(db)

        if settings.active_round != "round 1":

            raise HTTPException(
                status_code=403,
                detail="Round 1 is not active"
            )

        if team.status != "round 1":

            raise HTTPException(
                status_code=403,
                detail="Team is not eligible for Round 1"
            )

        attempt = get_or_create_attempt(
            db,
            team,
            "round 1"
        )

        assignments = db.query(
            models.TeamQuestionAssignment
        ).filter(
            models.TeamQuestionAssignment.team_id
            == team.team_id
        ).all()

        if not assignments:

            easy = db.query(
                models.Round1Question
            ).filter(
                models.Round1Question.level_of_difficulty
                == "easy"
            ).all()

            medium = db.query(
                models.Round1Question
            ).filter(
                models.Round1Question.level_of_difficulty
                == "medium"
            ).all()

            hard = db.query(
                models.Round1Question
            ).filter(
                models.Round1Question.level_of_difficulty
                == "hard"
            ).all()

            if (
                len(easy) < 20
                or len(medium) < 20
                or len(hard) < 20
            ):

                raise HTTPException(
                    status_code=400,
                    detail=(
                        "Question bank requires at least "
                        "20 easy, 20 medium and 20 hard questions"
                    )
                )

            selected = (
                random.sample(easy, 20)
                + random.sample(medium, 20)
                + random.sample(hard, 20)
            )

            random.shuffle(selected)

            for index, question in enumerate(
                selected,
                start=1
            ):

                assignment = (
                    models.TeamQuestionAssignment(
                        team_id=team.team_id,
                        question_id=question.id,
                        display_order=index
                    )
                )

                db.add(assignment)

            db.commit()

        return {
            "message": "Round 1 ready",
            "attempt_id": attempt.id,
            "started_at": attempt.started_at,
            "deadline": attempt.deadline,
            "final_submitted":
                attempt.final_submitted
        }

    finally:
        db.close()


# ==================================================
# ROUND 1 QUESTIONS FOR TEAM
# ==================================================

@app.get("/round1/questions")
def get_round1_questions(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        assignments = db.query(
            models.TeamQuestionAssignment
        ).filter(
            models.TeamQuestionAssignment.team_id
            == team.team_id
        ).order_by(
            models.TeamQuestionAssignment.display_order
        ).all()

        result = []

        for assignment in assignments:

            question = db.query(
                models.Round1Question
            ).filter(
                models.Round1Question.id
                == assignment.question_id
            ).first()

            if question:

                result.append({
                    "display_order":
                        assignment.display_order,

                    "id":
                        question.id,

                    "question_number":
                        question.question_number,

                    "question":
                        question.question,

                    "options":
                        question.options,

                    "pic":
                        question.pic,

                    "difficulty":
                        question.level_of_difficulty,

                    "marks":
                        question.marks
                })

        return {
            "questions": result
        }

    finally:
        db.close()


# ROUND 1 SAVE ANSWER

@app.post("/round1/answer")
def save_round1_answer(
    data: schemas.AnswerSave,
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        attempt = db.query(
            models.RoundAttempt
        ).filter(
            models.RoundAttempt.team_id
            == team.team_id,

            models.RoundAttempt.round
            == "round 1"
        ).first()

        if attempt is None:

            raise HTTPException(
                status_code=400,
                detail="Round 1 has not been started"
            )

        ensure_attempt_open(attempt)

        assignment = db.query(
            models.TeamQuestionAssignment
        ).filter(
            models.TeamQuestionAssignment.team_id
            == team.team_id,

            models.TeamQuestionAssignment.question_id
            == data.question_id
        ).first()

        if assignment is None:

            raise HTTPException(
                status_code=400,
                detail="Question is not assigned to this team"
            )

        question = db.query(
            models.Round1Question
        ).filter(
            models.Round1Question.id
            == data.question_id
        ).first()

        selected = data.selected_option_id

        if selected is not None:

            selected = selected.upper().strip()

            valid_options = {
                str(option.get("option_id", "")).upper()
                for option in question.options
            }

            if selected not in valid_options:

                raise HTTPException(
                    status_code=400,
                    detail="Invalid option"
                )

        submission = db.query(
            models.Submission
        ).filter(
            models.Submission.team_id
            == team.team_id,

            models.Submission.round
            == "round 1",

            models.Submission.question_id
            == data.question_id
        ).first()

        if submission is None:

            submission = models.Submission(
                team_id=team.team_id,
                round="round 1",
                question_id=data.question_id
            )

            db.add(submission)

        submission.selected_option_id = selected

        submission.marked_for_review = (
            data.marked_for_review
        )

        # Do not expose or finalize correctness here.
        submission.is_correct = None
        submission.score_awarded = 0

        db.commit()

        return {
            "message": "Answer saved",
            "question_id": data.question_id,
            "selected_option_id": selected,
            "marked_for_review":
                data.marked_for_review
        }

    finally:
        db.close()


# ROUND 1 SAVED ANSWERS


@app.get("/round1/answers")
def get_saved_round1_answers(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        submissions = db.query(
            models.Submission
        ).filter(
            models.Submission.team_id
            == team.team_id,

            models.Submission.round
            == "round 1"
        ).all()

        return {
            "answers": [
                {
                    "question_id":
                        submission.question_id,

                    "selected_option_id":
                        submission.selected_option_id,

                    "marked_for_review":
                        submission.marked_for_review
                }

                for submission in submissions
            ]
        }

    finally:
        db.close()


# ROUND 1 FINAL SUBMISSION + SCORING

@app.post("/round1/submit")
def submit_round1(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        attempt = db.query(
            models.RoundAttempt
        ).filter(
            models.RoundAttempt.team_id
            == team.team_id,

            models.RoundAttempt.round
            == "round 1"
        ).first()

        if attempt is None:

            raise HTTPException(
                status_code=400,
                detail="Round 1 has not been started"
            )

        existing_result = db.query(
            models.TeamResult
        ).filter(
            models.TeamResult.team_id
            == team.team_id,

            models.TeamResult.round
            == "round 1"
        ).first()

        if attempt.final_submitted:

            return {
                "message":
                    "Round 1 was already submitted",

                "submitted": True
            }

        assignments = db.query(
            models.TeamQuestionAssignment
        ).filter(
            models.TeamQuestionAssignment.team_id
            == team.team_id
        ).all()

        total_score = 0.0
        attempted = 0
        correct_count = 0

        for assignment in assignments:

            question = db.query(
                models.Round1Question
            ).filter(
                models.Round1Question.id
                == assignment.question_id
            ).first()

            submission = db.query(
                models.Submission
            ).filter(
                models.Submission.team_id
                == team.team_id,

                models.Submission.round
                == "round 1",

                models.Submission.question_id
                == assignment.question_id
            ).first()

            if (
                submission is None
                or submission.selected_option_id is None
            ):
                continue

            attempted += 1

            is_correct = (
                submission.selected_option_id.upper()
                == question.correct_answer_id.upper()
            )

            submission.is_correct = is_correct

            if is_correct:

                submission.score_awarded = question.marks

                total_score += question.marks

                correct_count += 1

            else:

                submission.score_awarded = 0.0

        now = datetime.utcnow()

        attempt.final_submitted = True
        attempt.submitted_at = now

        time_taken = int(
            (
                min(now, attempt.deadline)
                - attempt.started_at
            ).total_seconds()
        )

        if time_taken < 0:
            time_taken = 0

        if existing_result is None:

            existing_result = models.TeamResult(
                team_id=team.team_id,
                round="round 1"
            )

            db.add(existing_result)

        existing_result.total_score = total_score

        existing_result.questions_attempted = attempted

        existing_result.correct_count = correct_count

        existing_result.time_taken_seconds = time_taken

        existing_result.is_final_submitted = True

        db.commit()

        # Intentionally don't reveal score to participant here.
        return {
            "message":
                "Round 1 submitted successfully",

            "submitted": True
        }

    finally:
        db.close()


# ROUND 2 START


@app.post("/round2/start")
def start_round2(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        settings = get_settings(db)

        if settings.active_round != "round 2":

            raise HTTPException(
                status_code=403,
                detail="Round 2 is not active"
            )

        if team.status != "round 2":

            raise HTTPException(
                status_code=403,
                detail="Team is not eligible for Round 2"
            )

        attempt = get_or_create_attempt(
            db,
            team,
            "round 2"
        )

        return {
            "message": "Round 2 ready",
            "attempt_id": attempt.id,
            "started_at": attempt.started_at,
            "deadline": attempt.deadline
        }

    finally:
        db.close()



# ROUND 2 QUESTIONS

@app.get("/round2/questions")
def get_round2_questions(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        team = authenticate_team(
            db,
            x_access_token
        )

        if team.status != "round 2":

            raise HTTPException(
                status_code=403,
                detail="Team is not eligible for Round 2"
            )

        questions = db.query(
            models.Round2Question
        ).order_by(
            models.Round2Question.question_number
        ).all()

        return {
            "questions": [
                {
                    "id": q.id,
                    "question_number":
                        q.question_number,
                    "question":
                        q.question,
                    "pic":
                        q.pic,
                    "tool":
                        q.tool
                }

                for q in questions
            ]
        }

    finally:
        db.close()


# ADMIN RESULTS


@app.get("/admin/results")
def get_results(
    x_access_token: str = Header(...)
):

    db = SessionLocal()

    try:

        require_admin(
            db,
            x_access_token
        )

        results = db.query(
            models.TeamResult
        ).all()

        output = []

        for result in results:

            team = db.query(
                models.Team
            ).filter(
                models.Team.team_id
                == result.team_id
            ).first()

            output.append({
                "team_id":
                    result.team_id,

                "team_name":
                    team.team_name
                    if team
                    else None,

                "round":
                    result.round,

                "total_score":
                    result.total_score,

                "questions_attempted":
                    result.questions_attempted,

                "correct_count":
                    result.correct_count,

                "time_taken_seconds":
                    result.time_taken_seconds,

                "is_final_submitted":
                    result.is_final_submitted
            })

        return {
            "results": output
        }

    finally:
        db.close()
#database.py
import os

from sqlalchemy import create_engine
from sqlalchemy.orm import declarative_base, sessionmaker


DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "sqlite:///./electronica.db"
)

connect_args = {}

if DATABASE_URL.startswith("sqlite"):
    connect_args = {"check_same_thread": False}


engine = create_engine(
    DATABASE_URL,
    connect_args=connect_args
)


SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)


Base = declarative_base()

#models.py
import uuid

from sqlalchemy import (
    Column,
    String,
    Integer,
    Float,
    Boolean,
    DateTime,
    Text,
    ForeignKey,
    UniqueConstraint,
    JSON
)

from datetime import datetime

from database import Base


def uuid_string():
    return str(uuid.uuid4())


# ==================================================
# TEAMS
# ==================================================

class Team(Base):
    __tablename__ = "teams"

    team_id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    team_name = Column(
        String(120),
        nullable=False
    )

    role = Column(
        String(20),
        default="team",
        nullable=False
    )

    access_token = Column(
        String(64),
        unique=True,
        nullable=False
    )

    status = Column(
        String(20),
        default="round 1",
        nullable=False
    )

    created_at = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False
    )



# COMPETITION SETTINGS


class CompetitionSettings(Base):
    __tablename__ = "competition_settings"

    id = Column(
        Integer,
        primary_key=True,
        default=1
    )

    active_round = Column(
        String(20),
        default="round 1",
        nullable=False
    )

    round_duration_minutes = Column(
        Integer,
        default=45,
        nullable=False
    )

    updated_at = Column(
        DateTime,
        default=datetime.utcnow,
        onupdate=datetime.utcnow,
        nullable=False
    )


# ROUND 1 QUESTIONS

class Round1Question(Base):
    __tablename__ = "round1_questions"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    question_number = Column(
        Integer,
        nullable=False
    )

    question = Column(
        Text,
        nullable=False
    )

    options = Column(
        JSON,
        nullable=False
    )

    pic = Column(
        Text,
        nullable=True
    )

    correct_answer_id = Column(
        String(10),
        nullable=False
    )

    level_of_difficulty = Column(
        String(20),
        nullable=False
    )

    marks = Column(
        Float,
        default=1.0,
        nullable=False
    )



# ROUND 2 QUESTIONS


class Round2Question(Base):
    __tablename__ = "round2_questions"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    question_number = Column(
        Integer,
        nullable=False
    )

    question = Column(
        Text,
        nullable=False
    )

    pic = Column(
        Text,
        nullable=True
    )

    tool = Column(
        String(100),
        nullable=False
    )

    evaluation_criteria = Column(
        Text,
        nullable=True
    )


# TEAM ROUND-1 QUESTION ASSIGNMENT


class TeamQuestionAssignment(Base):
    __tablename__ = "team_question_assignments"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    team_id = Column(
        String(36),
        ForeignKey("teams.team_id"),
        nullable=False
    )

    question_id = Column(
        String(36),
        ForeignKey("round1_questions.id"),
        nullable=False
    )

    display_order = Column(
        Integer,
        nullable=False
    )

    assigned_at = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False
    )

    __table_args__ = (
        UniqueConstraint(
            "team_id",
            "question_id",
            name="uq_team_question"
        ),
        UniqueConstraint(
            "team_id",
            "display_order",
            name="uq_team_display_order"
        ),
    )

# ROUND ATTEMPTS

class RoundAttempt(Base):
    __tablename__ = "round_attempts"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    team_id = Column(
        String(36),
        ForeignKey("teams.team_id"),
        nullable=False
    )

    round = Column(
        String(20),
        nullable=False
    )

    started_at = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False
    )

    deadline = Column(
        DateTime,
        nullable=False
    )

    final_submitted = Column(
        Boolean,
        default=False,
        nullable=False
    )

    submitted_at = Column(
        DateTime,
        nullable=True
    )

    __table_args__ = (
        UniqueConstraint(
            "team_id",
            "round",
            name="uq_team_round_attempt"
        ),
    )


# SUBMISSIONS / AUTOSAVED ANSWERS

class Submission(Base):
    __tablename__ = "submissions"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    team_id = Column(
        String(36),
        ForeignKey("teams.team_id"),
        nullable=False
    )

    round = Column(
        String(20),
        nullable=False
    )

    question_id = Column(
        String(36),
        nullable=False
    )

    selected_option_id = Column(
        String(10),
        nullable=True
    )

    marked_for_review = Column(
        Boolean,
        default=False,
        nullable=False
    )

    is_correct = Column(
        Boolean,
        nullable=True
    )

    score_awarded = Column(
        Float,
        default=0.0,
        nullable=False
    )

    submitted_at = Column(
        DateTime,
        default=datetime.utcnow,
        onupdate=datetime.utcnow,
        nullable=False
    )

    __table_args__ = (
        UniqueConstraint(
            "team_id",
            "round",
            "question_id",
            name="uq_team_round_question_submission"
        ),
    )
# TEAM RESULTS

class TeamResult(Base):
    __tablename__ = "team_results"

    id = Column(
        String(36),
        primary_key=True,
        default=uuid_string
    )

    team_id = Column(
        String(36),
        ForeignKey("teams.team_id"),
        nullable=False
    )

    round = Column(
        String(20),
        nullable=False
    )

    total_score = Column(
        Float,
        default=0.0,
        nullable=False
    )

    questions_attempted = Column(
        Integer,
        default=0,
        nullable=False
    )

    correct_count = Column(
        Integer,
        default=0,
        nullable=False
    )

    time_taken_seconds = Column(
        Integer,
        default=0,
        nullable=False
    )

    is_final_submitted = Column(
        Boolean,
        default=False,
        nullable=False
    )

    __table_args__ = (
        UniqueConstraint(
            "team_id",
            "round",
            name="uq_team_round_result"
        ),
    )
#Schemas.py
from typing import Optional, List

from pydantic import BaseModel, Field


class TeamCreate(BaseModel):
    team_name: str
    access_token: str
    role: str = "team"
    status: str = "round 1"


class LoginRequest(BaseModel):
    access_token: str


class Round1QuestionCreate(BaseModel):
    question_number: int
    question: str

    options: List[dict]

    pic: Optional[str] = None

    correct_answer_id: str

    level_of_difficulty: str

    marks: float = 1.0


class Round2QuestionCreate(BaseModel):
    question_number: int
    question: str

    pic: Optional[str] = None

    tool: str

    evaluation_criteria: Optional[str] = None


class AnswerSave(BaseModel):
    question_id: str

    selected_option_id: Optional[str] = None

    marked_for_review: bool = False


class RoundSwitch(BaseModel):
    active_round: str

    round_duration_minutes: Optional[int] = Field(
        default=None,
        ge=1,
        le=300
    )


class TeamStatusUpdate(BaseModel):
    status: str
